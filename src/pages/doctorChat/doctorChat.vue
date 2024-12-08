<template>
  <view class="container">
    <view class="chat-header">
      <text class="doctor-name">{{ doctor.name }}</text>
    </view>
    <scroll-view class="chat-body" scroll-y="true" :scroll-top="scrollTop" @scrolltoupper="loadMoreMessages"
      @scrolltolower="loadLatestMessages" :scroll-with-animation="true" :refresher-enabled="true"
      :refresher-triggered="isRefreshing" @refresherrefresh="onRefresh">
      <view v-for="message in messages" :key="message.messageSeq" class="message-wrapper"
        :class="{ 'self-message': message.isSelf }">
        <image v-if="!message.isSelf" :src="doctor.avatarUrl" class="avatar"></image>
        <view class="message-content" :class="{ 'self-content': message.isSelf }">
          <image v-if="message.type === 'image'" :src="message.localUrl || message.url" mode="widthFix"
            class="message-image" alt="Message image"></image>
          <text v-else class="message-text">{{ message.content }}</text>
        </view>
        <image v-if="message.isSelf" :src="userAvatar" class="avatar"></image>
      </view>

    </scroll-view>
    <view class="chat-footer">
      <input v-model="inputMessage" class="message-input" type="text" placeholder="输入消息..." @confirm="sendMessage" />
      <button class="send-button" @tap="sendMessage">发送</button>
      <button class="image-button" @tap="chooseImage">图片</button>
    </view>
  </view>
</template>

<script>
import { selectRelationIdByDoctorId } from '@/api/relation';
import {
  getLast30Messages,
  getMessagesBefore,
  getMessagesAfter,
  sendMessageApi,
  updateReadInfo
} from '@/api/message';
import { getDoctorAvatar, uploadChatImageApi } from '@/api/image';

export default {
  data() {
    return {
      doctor: {
        doctorId: '',
        name: '',
        avatarUrl: '',
        workplace: '',
        rating: 0,
        position: '',
        experience: ''
      },
      relationId: '',
      messages: [],
      inputMessage: '',
      scrollTop: 0,
      userAvatar: '',
      isRefreshing: false,
      messageSeqBefore: null, // 用于加载历史消息
      messageSeqAfter: null,  // 用于加载最新消息
      loadingMore: false,     // 防止重复加载
      loadingLatest: false,   // 防止重复加载
    };
  },
  async onLoad(option) {
    try {
      if (option.doctor) {
        this.doctor = JSON.parse(decodeURIComponent(option.doctor));
      }
      this.relationId = await selectRelationIdByDoctorId(this.doctor.doctorId);
      await this.fetchMessages();
      this.scrollToBottom(); // 初始化完成后滚动到底部
    } catch (error) {
      console.error('页面初始化失败:', error);
      uni.showToast({
        title: '初始化失败，请稍后重试',
        icon: 'none',
      });
    }
  },

  methods: {
    // 初始加载消息
    async fetchMessages() {
      try {
        console.log('开始加载初始化消息');
        const messages = await getLast30Messages(this.relationId);
        console.log('初始消息数据:', messages); // 日志

        const processedMessages = await this.processMessages(messages);
        console.log('处理后的消息:', processedMessages); // 日志

        this.messages = processedMessages.sort((a, b) => a.messageSeq - b.messageSeq);

        // 初始化 messageSeqBefore 和 messageSeqAfter
        this.messageSeqBefore = this.messages[0]?.messageSeq || null;
        this.messageSeqAfter = this.messages[this.messages.length - 1]?.messageSeq || null;

        console.log('messageSeqBefore:', this.messageSeqBefore, 'messageSeqAfter:', this.messageSeqAfter);
        // 在成功加载最新消息后，更新已读消息序列号
        if (this.messageSeqAfter) {
          try {
            await updateReadInfo(this.relationId, this.messageSeqAfter);
            console.log('已更新已读消息序列号:', this.messageSeqAfter);
          } catch (error) {
            console.error('更新已读消息序列号失败:', error);
          }
        }
        this.scrollToBottom();
      } catch (error) {
        console.error('加载消息失败:', error);
        uni.showToast({
          title: '加载消息失败，请稍后重试',
          icon: 'none',
        });
      }
    },

    async chooseImage() {
      uni.chooseImage({
        count: 1,
        success: async (res) => {
          const tempFilePath = res.tempFilePaths[0]; // 获取临时图片路径

          // 创建临时图片消息对象
          const newImageMessage = { // 临时路径
            localUrl: tempFilePath,
            isSelf: true,
            type: 'image',
            url: null, // 后端返回的 URL
          };

          // 添加到消息列表中
          this.messages.push(newImageMessage);
          this.$nextTick(() => {
            this.scrollToBottom();
          });

          try {
            // 上传图片到服务器
            const response = await uploadChatImageApi(tempFilePath, this.relationId);

            if (response && response.imageUrl) {
              // 更新消息对象的 URL 和内容
              newImageMessage.url = response.imageUrl;
              //newImageMessage.content = response.imageUrl;

              // 不再发送 `[图片]` 文本消息，而是更新消息状态到后端
              await sendMessageApi(
                this.relationId,
                'user',
                '[图片]', // 后端存储类型标识
                'image',
                response.imageUrl
              );

              console.log('图片发送成功:', response.imageUrl);
            } else {
              throw new Error('图片上传失败，未返回 URL');
            }
          } catch (error) {
            console.error('图片发送失败:', error);
            uni.showToast({
              title: '图片发送失败，请稍后重试',
              icon: 'none',
            });

            // 移除发送失败的临时消息
            const index = this.messages.indexOf(newImageMessage);
            if (index !== -1) {
              this.messages.splice(index, 1);
            }
          }
        },
        fail: (err) => {
          console.error('选择图片失败:', err);
          uni.showToast({
            title: '选择图片失败，请稍后重试',
            icon: 'none',
          });
        },
      });
    },
    // 处理消息数据
    async processMessages(messages) {
      return Promise.all(
        messages
          .filter((msg) => msg && msg.messageSeq) // 确保消息有效并包含 messageSeq
          .map(async (msg) => {
            let url = msg.url;
            if (msg.messageType === 'image' && msg.url) {
              try {
                url = await getDoctorAvatar(msg.url); // 默认使用 URL
              } catch (error) {
                console.error('加载图片失败:', error);
              }
            }
            return {
              content: msg.messageText,
              isSelf: msg.senderType === 'user',
              url: url,
              type: msg.messageType,
              messageSeq: msg.messageSeq, // 确保 messageSeq 正确设置
            };
          })
      );
    },


    // 发送消息
    async sendMessage() {
      if (!this.inputMessage.trim()) return;

      const newMessage = {
        content: this.inputMessage,
        isSelf: true,
        type: 'text',
        messageSeq: this.messageSeqAfter + 1 || 1, // 假设从1开始递增
      };

      this.messages.push(newMessage);
      this.inputMessage = '';

      this.$nextTick(() => {
        this.scrollToBottom();
      });

      try {
        const response = await sendMessageApi(this.relationId, 'user', newMessage.content);

        // 更新最新的消息序列号
        this.messageSeqAfter = newMessage.messageSeq;
        console.log('消息发送成功，更新消息序列号:', this.messageSeqAfter);

        // 可能需要后端返回一个确认的序列号，确保一致性
        if (response && response.messageSeq) {
          this.messageSeqAfter = response.messageSeq;
          console.log('消息发送成功，后端返回的消息序列号:', response.messageSeq);
        }
      } catch (error) {
        console.error('发送消息失败:', error);
        uni.showToast({
          title: '消息发送失败，请稍后重试',
          icon: 'none',
        });

        // 失败后移除临时消息
        const index = this.messages.indexOf(newMessage);
        if (index !== -1) {
          this.messages.splice(index, 1);
        }
      }
    },


    // 加载更多历史消息
    async loadMoreMessages() {
      if (this.loadingMore || !this.messageSeqBefore) return;
      this.loadingMore = true;

      console.log('开始加载历史消息', this.messageSeqBefore);

      try {
        const moreMessages = await getMessagesBefore(this.relationId, this.messageSeqBefore)

        if (moreMessages.length === 0) {
          console.log('没有更多历史消息');
          return; // 如果为空，直接退出
        }

        const processedMessages = await this.processMessages(moreMessages[1]);
        console.log('处理后的历史消息:', processedMessages);

        this.messages = processedMessages
          .sort((a, b) => a.messageSeq - b.messageSeq)
          .concat(this.messages);

        this.messageSeqBefore = this.messages[0]?.messageSeq || this.messageSeqBefore; // 更新序列号
        console.log('更新后的历史消息:', this.messages);
      } catch (error) {
        console.error('加载历史消息失败:', error);
        uni.showToast({
          title: '加载历史消息失败，请稍后重试',
          icon: 'none',
        });
      } finally {
        this.loadingMore = false;
      }
    },

    async loadLatestMessages() {
      console.log('触发了加载最新消息', this.messageSeqAfter);
      if (this.loadingLatest || !this.messageSeqAfter) return;
      console.log('开始加载最新消息');
      this.loadingLatest = true;

      try {
        const latestMessages = await getMessagesAfter(this.relationId, this.messageSeqAfter);

        if (latestMessages.length === 0) {
          console.log('没有新消息');
          return;
        }

        // 处理消息数据
        const processedMessages = await this.processMessages(latestMessages[1]);

        // 过滤掉已加载的消息
        const newMessages = processedMessages.filter(
          (msg) => msg.messageSeq > this.messageSeqAfter
        );

        if (newMessages.length > 0) {
          this.messages = this.messages.concat(
            newMessages.sort((a, b) => a.messageSeq - b.messageSeq)
          );

          this.messageSeqAfter = this.messages[this.messages.length - 1]?.messageSeq || this.messageSeqAfter;
          console.log('加载最新消息成功');

          // 更新已读消息序列号
          if (this.messageSeqAfter) {
            try {
              await updateReadInfo(this.relationId, this.messageSeqAfter);
              console.log('已更新已读消息序列号:', this.messageSeqAfter);
            } catch (error) {
              console.error('更新已读消息序列号失败:', error);
            }
          }
        } else {
          console.log('没有新的消息需要添加');
        }
      } catch (error) {
        console.error('加载最新消息失败:', error);
      } finally {
        this.loadingLatest = false;
      }
    },


    // 滚动到底部
    scrollToBottom() {
      this.$nextTick(() => {
        const query = uni.createSelectorQuery().in(this);
        query.select('.chat-body').boundingClientRect();
        query.selectViewport().scrollOffset();
        query.exec((res) => {
          if (res[0] && res[1]) {
            this.scrollTop = res[0].height;
          }
        });
      });
    },

    async onRefresh() {
      console.log("触发刷新操作，加载历史消息");
      this.isRefreshing = true; // 开启刷新状态

      try {
        await this.fetchMessages()
      } catch (error) {
        console.error("刷新时加载历史消息失败:", error);
        uni.showToast({
          title: "加载历史消息失败，请稍后重试",
          icon: "none",
        });
      } finally {
        this.isRefreshing = false; // 关闭刷新状态
      }
    },


    // 图片预览
    previewImage(url) {
      uni.previewImage({
        urls: [url],
        current: url,
      });
    },
  },
};
</script>


<style scoped>
.container {
  display: flex;
  flex-direction: column;
  height: 100vh;
}

.chat-header {
  background-color: #f8f8f8;
  padding: 10px;
  text-align: center;
  border-bottom: 1px solid #e0e0e0;
}

.doctor-name {
  font-size: 18px;
  font-weight: bold;
}

.chat-body {
  flex: 1;
  padding: 10px;
  background-color: #f0f0f0;
  overflow-y: auto;
  /* 避免滚动问题 */
}

.message-wrapper {
  display: flex;
  margin-bottom: 10px;
}

.self-message {
  justify-content: flex-end;
}

.avatar {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  margin: 0 10px;
}

.message-content {
  max-width: 70%;
  padding: 10px;
  border-radius: 10px;
  background-color: #fff;
}

.self-content {
  background-color: #9eea6a;
}

.message-text {
  font-size: 16px;
  line-height: 1.4;
}

.message-image {
  max-width: 100%;
  border-radius: 5px;
}

.chat-footer {
  display: flex;
  padding: 10px;
  background-color: #f8f8f8;
  border-top: 1px solid #e0e0e0;
}

.message-input {
  flex: 1;
  height: 36px;
  padding: 0 10px;
  border: 1px solid #ddd;
  border-radius: 18px;
  font-size: 16px;
}

.send-button,
.image-button {
  margin-left: 10px;
  padding: 0 15px;
  height: 36px;
  line-height: 36px;
  border: none;
  border-radius: 18px;
  background-color: #007aff;
  color: #fff;
  font-size: 16px;
}

.image-button {
  background-color: #4cd964;
}
</style>