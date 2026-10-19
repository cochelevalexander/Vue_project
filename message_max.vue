<template> <!-- Диалоговое окно Маха, -->
  <div>
    <div class="max-icon" @click="showChat">
      <img src="max1.png" alt="MAX Messenger" width="86" height="48">
    </div>

    <div v-if="interlocutor" class="chat-modal">
      <div class="chat-container">
        <div class="chat-header">
          <div class="header-info">
            <span class="chat-title">MAX Messenger</span>
            <span v-if="userInfo" class="user-name">{{ userInfo.name }}</span>
          </div>
          <button class="close-btn" @click="closeChat">×</button>
        </div>

        <div class="input-group">
          <input
            v-model="chatID"
            placeholder="Введите ID пользователя"
            class="input-field"
            @keyup.enter="requestInfo"
          >
          <button @click="requestInfo" class="send-button" :disabled="loading">
            Получить диалог
          </button>
        </div>

        <div class="messages-area" ref="messagesContainer">
          <div v-if="!response || !response.messages || response.messages.length === 0" class="empty-messages">
            <p>Нет сообщений</p>
            <small>Введите ID и нажмите "Получить диалог"</small>
          </div>

          <div v-if="response && response.messages && response.messages.length > 0">
            <div 
              v-for="(message, index) in [...response.messages].reverse()" 
              :key="index" 
              class="message-item"
            >
              <div :class="['message-bubble', getSenderName(message) === 'Mixtell' ? 'own' : 'other']">
              <div class="message-sender">{{ getSenderName(message) }}</div>
              <div v-if="getMessageText(message) !== '' " class="message-text">{{ getMessageText(message) }}</div>
              <div v-if="getImageUrl(message)" class="message-image"><img :src="getImageUrl(message)" class="attachment-image">
        </div>
              <div class="message-time">{{ formatTime(message.timestamp) }}</div>
            </div>
            </div>
          </div>
        </div>

        <div class="input-area">
          <div v-if="attachedFile" class="attachment-preview">
            <div class="file-info">
              <span class="file-icon">📎</span>
              <span class="file-name">{{ attachedFile.name }}</span>
            </div>
            <div v-if="attachmentType === 'image'" class="image-preview">
            </div>
          </div>

          <div class="input-controls">
            <label class="attach-btn">
              <img src="needle.png" alt="MAX Messenger" width="32" height="24">
              <input
                type="file"
                @change="onChange"
                accept="image/*,video/*,audio/*,.doc,.docx,.xls,.xlsx,.pdf,.txt"
                style="display: none"
              >
            </label>

            <textarea
              v-model="newMessage"
              placeholder="Введите сообщение..."
              class="message-input"
              rows="1"
              @keyup.enter.prevent="sendMessage"
            ></textarea>
            
            <button @click="sendMessage" class="send-message-btn" :disabled="!newMessage.trim() && !attachedFile">
              Отправить
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
<script>
import axios from 'axios';

export default {
  name: 'My-messangeMax',
  data() {
    return {
      interlocutor: false,
      chatID: null,
      authToken: null,
      loading: false,
      error: null,
      response: null,
      userInfo: null,
      newMessage: '',
      imageUri: '',
      attachmentType: null,
      attachmentPayload: null,
      attachedFileName: null,
      attachedUri: null,
      file: null,
    }
  },
  watch: {
    response: {
      handler() {
        this.$nextTick(() => {
          this.scrollToBottom();
        });
      },
      deep: true
    }
  },
  mounted() {
    this.intervalId = setInterval(() => {
    if (this.interlocutor && this.chatID && this.loading) {  
      this.requestInfo();
    }
  }, 7000);
  },
  beforeUnmount() {
    if (this.intervalId) {
      clearInterval(this.intervalId);
    }
  },
  methods: {
    previewImage(url) {
    window.open(url, '_blank');
  },
    showChat() {
      this.interlocutor = true;
      if (this.interlocutor && this.chatID){
         this.requestInfo()
      }
    },
    async onChange(e) {
      console.log('Отправка файла на сервер')
      this.attachedFile = e.target.files[0];
      this.attachedFileName = this.attachedFile.name
      const ext = this.attachedFile.name.split('.').pop().toLowerCase()
      if (ext === 'png' || ext === 'jpg' || ext === 'jpeg' || ext === 'gif'){
        this.attachmentType = 'image'
        console.log('Изображение')
      }
      else{
        this.attachmentType = 'file'
        console.log('Файл')
      }
    try {
      if (!this.authToken) {
        await this.getAuthToken();
      }
      const formDataFile = new FormData();
      formDataFile.append('attachedFile', this.attachedFile);
      formDataFile.append('attachedFileName', this.attachedFileName);
      formDataFile.append('attachmentType', this.attachmentType);
      console.log('Дата файл', formDataFile)
      const { data } = await axios.post('/api/send_max_file', formDataFile,
      {
        headers: { 
          Authorization: `Bearer ${this.authToken}` 
        }
      }
      
    );
    console.log('Получен токен:', data)
    if (data.success) {
  const fullResponse = JSON.parse(data.full_response);
  
  let token = null;

  if (fullResponse.photos) {
    for (let key in fullResponse.photos) {
      if (fullResponse.photos[key].token) {
        token = fullResponse.photos[key].token;
        break;
      }
    }
  }

  else if (fullResponse.token) {
    token = fullResponse.token;
  }

  else if (fullResponse.fileId) {
    token = fullResponse.fileId.toString();
  }
  
  if (token) {
    this.attachmentToken = token;
    console.log('Токен извлечен:', this.attachmentToken);
  } else {
    this.attachmentUri = data.url;
    console.log('URL сохранен:', this.attachmentUri);
  }
  }

    }catch (err) {
    console.error('Ошибка загрузки файла:', err);
    }
      
    },
    async sendMessage() {
      console.log('вход в отправку')
  if ((!this.newMessage || !this.newMessage.trim()) && !this.attachmentToken) return;
  if (!this.chatID) return;
  
  if (!this.authToken) {
    await this.getAuthToken();
    if (this.error) return;
  }
  
  try {
    console.log(this.chatID)
    console.log(this.newMessage )
    const formData = {
      chatID: this.chatID,
      newMessage: this.newMessage,
  };
    if (this.attachmentToken) {
      formData.attachment = this.attachmentToken;
      formData.attachmentType = this.attachmentType; 
    }
    console.log('Отправить сообщение')
    console.log('Текст', this.newMessage)
    console.log('файл', this.attachmentToken)
    const { data } = await axios.post('/api/send_max_messange', formData,
      {
        headers: { 
          Authorization: `Bearer ${this.authToken}` 
        }
      }
    );
    
    
    if (data.success) {
      this.newMessage = '';
      this.attachedFile = null; 
      this.attachmentType = null;
      this.attachedUri = null;
    } else {
      console.error('Ошибка:', data.error);
    }
    
  } catch (err) {
    console.error('Ошибка отправки:', err);
  }
},

    closeChat() {
      this.interlocutor = false;
      this.response = null;
      this.chatID = null;
      this.userInfo = null;
    },
    
    scrollToBottom() {
      const container = this.$refs.messagesContainer;
      if (container) {
        container.scrollTop = container.scrollHeight;
      }
    },
    getAttachments(message) {
    return message.body?.attachments || message.attachments || [];
  },
    
    async getAuthToken() {
      try {
        const { data } = await axios.post('/api/login', {
          login: this.username,
          password: this.password
        });
        this.authToken = data.token;
      } catch (err) {
        this.error = 'Ошибка авторизации';
        console.error('Auth error:', err);
      }
    },

    async requestInfo() {
      if (!this.authToken) {
        await this.getAuthToken();
        if (this.error) return;
      }
      
      console.log('Вход в функцию');
      console.log(this.chatID);
      this.loading = true;
      this.error = null;
      
      try {
        const { data } = await axios.get('/api/max_message', {
          params: { chatID: this.chatID },
          headers: { 
            Authorization: `Bearer ${this.authToken}` 
          },
          timeout: 30000
        });

        this.response = data;
        console.log('Получен ответ:', this.response);

        if (this.response && this.response.messages && this.response.messages.length > 0) {
          const firstMessage = this.response.messages[0];
          if (firstMessage && firstMessage.sender) {
            this.userInfo = firstMessage.sender;
          }
        }
        
      } catch (err) {
        this.error = 'Ошибка получения данных';
        console.error('Request error:', err);
        this.response = null;
      } finally {
        this.loading = false;
      }
    },
    
    getMessageText(message) {
  if (message.body?.text) return message.body.text;
  if (message.text) return message.text;
  return '';
},

getImageUrl(message) {
  const attachments = message.body?.attachments || message.attachments || [];
  
  for (const attachment of attachments) {
    if (attachment.type === 'sticker' || attachment.type === 'image') {
      return attachment.payload?.url || attachment.url;
    }
  }
  return null;
},

  getFirstAttachmentUrl(message) {
    const attachments = message.body?.attachments || message.attachments || [];
    const imageTypes = ['sticker', 'image'];
    
    for (const attachment of attachments) {
      if (imageTypes.includes(attachment.type) && attachment.url) {
        return attachment.url;
      }
    }
    return null;
  },

  hasImageAttachment(message) {
    const attachments = message.body?.attachments || message.attachments || [];
    const imageTypes = ['sticker', 'image'];
    return attachments.some(att => imageTypes.includes(att.type));
  },
    
    getSenderName(message) {
      if (message.sender) {
        return message.sender.name || message.sender.first_name || `User ${message.sender.user_id}`;
      }
      return 'Пользователь';
    },
    
    isOwnMessage(message) {
      const currentUserId = this.$store?.getters?.getCredentials?.userId;
      if (currentUserId && message.sender) {
        return message.sender.user_id === currentUserId;
      }
      return false;
    },
    
    formatTime(timestamp) {
      if (!timestamp) return '';
      const date = new Date(timestamp);
      return date.toLocaleString('ru-RU', {
        hour: '2-digit',
        minute: '2-digit',
        day: '2-digit',
        month: '2-digit'
      });
    }
  }
}
</script>

<style>
.max-icon {
  bottom: 30px;
  right: 80px;
  cursor: pointer;
  z-index: 1000;
  transition: transform 0.2s;
}

.max-icon:hover {
  transform: scale(1.02);
}

.max-icon img {
  display: block;
}

.chat-modal {
  position: fixed;
  bottom: 100px;
  right: 30px;
  width: 60%;
  height: 80%;
  background: white;
  border-radius: 12px;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.2);
  z-index: 11;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.chat-container {
  display: flex;
  flex-direction: column;
  height: 100%;
}

.chat-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px 16px;
  background: linear-gradient(135deg, #0b75ee, rgb(163, 76, 221));
  color: white;
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
}

.header-info {
  display: flex;
  flex-direction: column;
}

.chat-title {
  font-weight: bold;
  font-size: 16px;
}

.user-name {
  font-size: 12px;
  opacity: 0.9;
}

.close-btn {
  background: none;
  border: none;
  color: white;
  font-size: 24px;
  cursor: pointer;
  padding: 0;
  width: 30px;
  height: 30px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 50%;
}

.close-btn:hover {
  background: rgba(255, 255, 255, 0.2);
}

.input-group {
  display: flex;
  gap: 8px;
  padding: 12px;
  border-bottom: 1px solid #e0e0e0;
}

.input-field {
  flex: 1;
  padding: 8px 12px;
  border: 1px solid #ddd;
  border-radius: 6px;
  font-size: 14px;
}

.send-button {
  padding: 8px 16px;
  background: linear-gradient(135deg, #0b75ee, rgb(163, 76, 221));
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 14px;
}

.send-button:hover:not(:disabled) {
  background: linear-gradient(135deg, #0b75ee, rgb(163, 76, 221));
}

.send-button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.messages-area {
  flex: 1;
  overflow-y: auto;
  padding: 16px;
  background: #f5f5f5;
}

.empty-messages {
  text-align: center;
  color: #999;
  padding: 40px 20px;
}

.message-item {
  margin-bottom: 16px;
  display: flex;
}

.message-bubble {
  max-width: 70%;
  padding: 8px 12px;
  border-radius: 12px;
  word-wrap: break-word;
}

.message-bubble.own {
  background: linear-gradient(135deg, #0b75ee, rgb(163, 76, 221));
  color: white;
  margin-left: auto;
}

.message-bubble.other {
  background: white;
  color: #333;
  margin-right: auto;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
}

.message-sender {
  font-size: 12px;
  font-weight: bold;
  margin-bottom: 4px;
  opacity: 0.8;
}

.message-text {
  font-size: 14px;
  line-height: 1.4;
}

.message-time {
  font-size: 10px;
  text-align: right;
  margin-top: 4px;
  opacity: 0.6;
}

.input-area {
  gap: 8px;
  padding: 12px;
  border-top: 1px solid #e0e0e0;
  background: white;
  align-items: flex-end;
}

.attachment-buttons {
  display: flex;
  gap: 4px;
}

.attach-btn {
  background: none;
  border: none;
  font-size: 20px;
  cursor: pointer;
  padding: 8px;
  border-radius: 50%;
  width: 16px;
  height: 16px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.attach-btn:hover {
  background: #f0f0f0;
}

.message-input {
  flex: 1;
  padding: 8px 12px;
  border: 1px solid #ddd;
  border-radius: 18px;
  font-size: 14px;
  resize: none;
  font-family: inherit;
  max-height: 100px;
}

.send-message-btn {
  padding: 8px 16px;
  background: linear-gradient(135deg, #0b75ee, rgb(163, 76, 221));
  color: white;
  border: none;
  border-radius: 18px;
  cursor: pointer;
  font-size: 14px;
}

.send-message-btn:hover {
 background: linear-gradient(135deg, #0b75ee, rgb(163, 76, 221));
  
}
.message-image {
  margin-top: 8px;
}

.attachment-image {
  max-width: 200px;
  max-height: 200px;
  border-radius: 8px;
  cursor: pointer;
}

.attachment-image:hover {
  opacity: 0.9;
}
.attachment-preview {
  background: #f5f5f5;
  border-radius: 8px;
  padding: 10px 12px;
  margin-bottom: 10px;
  border: 1px solid #e0e0e0;
}

.file-info {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 5px;
}

.file-icon {
  font-size: 18px;
}

.file-name {
  font-weight: 500;
  color: #333;
  flex: 1;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.file-size {
  font-size: 11px;
  color: #888;
}

.remove-file-btn {
  background: none;
  border: none;
  font-size: 16px;
  cursor: pointer;
  color: #999;
  padding: 0 6px;
  border-radius: 4px;
  transition: all 0.2s;
}

.remove-file-btn:hover {
  color: #f44336;
  background: rgba(0, 0, 0, 0.05);
}

.image-preview {
  margin-top: 8px;
  padding-top: 8px;
  border-top: 1px solid #e0e0e0;
}

.preview-image {
  max-width: 100%;
  max-height: 120px;
  border-radius: 4px;
  display: block;
}

.input-controls {
  display: flex;
  gap: 10px;
  align-items: flex-end;
}

.attach-btn {
  background: #f0f0f0;
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 10px 14px;
  cursor: pointer;
  font-size: 18px;
  transition: all 0.2s;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  line-height: 1;
}

.attach-btn:hover {
  background: #e8e8e8;
  border-color: #ccc;
}

.attach-btn:active {
  transform: scale(0.98);
}

</style>