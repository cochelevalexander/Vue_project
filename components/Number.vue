<template> <!-- Компонент для запроса данных о клиенте в базу. Много лишнего, нужно профильтровать, так как изначально тут же была отрисовка из customer_card.vue-->
  <div class="phone-numbers-container">
    <div class="numbers-header">
      <div class="search-section">
        <input
          v-model="searchPhone"
          placeholder="Введите номер телефона"
          class="phone-input"
          @keyup.enter="requestInfo(searchPhone)"
        />
        <button @click="requestInfo(searchPhone)" class="search-btn" :disabled="loading">
          Поиск
        </button>
      </div>
    </div>
  </div>
</template>

<script>
import axios from 'axios';

export default {
  name: 'OpenNumber',
  data() {
    return {
      searchPhone: '',
      loading: false,
      error: null,
      request: null
    };
  },
  props: {
    project: String,
    number: String,
  },
  computed: {
  hasData() {
    return this.request && 
           this.request.success && 
           this.request.data && 
           this.request.data.model && 
           this.request.data.model.dealsObjects;
  },
  hasDataIpoteca() {
    return this.request
  }
},watch: {
  number() {
      if (this.number) {
        this.requestInfo(this.number)
      }
  },
},
  methods: {
    async getAuthToken() {
      try {
        const { data } = await axios.post('/api/login', {
          headers: { Authorization: `Bearer ${this.authToken}` }
        });
        this.authToken = data.token;
      } catch (err) {
        this.error = 'Ошибка авторизации';
        console.error('Auth error:', err);
      }
    },


    async requestInfo(phoneNumber) {
      if (!this.authToken) {
        await this.getAuthToken();
        if (this.error) return;
      }

      this.loading = true;
      this.error = null;
      
      try {
        let response;
        
        if (this.project === 'КАСКО' || this.project === 'КАСКО2') {
          const { data } = await axios.get('/api/base/kasko', {
            params: { requestId: phoneNumber },
            headers: { 
              Authorization: `Bearer ${this.authToken}` 
            },
            timeout: 30000
          });
          response = data;
          
        } else if (this.project === 'Ипотека') {
          const { data } = await axios.get('/api/base/ipoteka', {
            params: { requestId: phoneNumber },
            headers: { 
              Authorization: `Bearer ${this.authToken}` 
            },
            timeout: 30000
          });
          response = data;
        }
        
        this.request = response;
        this.$emit('request-updated', {
          request: this.request,
          project: this.project,
          phoneNumber: phoneNumber,
          timestamp: new Date().toISOString()
        });
        
      } catch (err) {
        this.error = 'Ошибка получения данных';
        console.error('Request error:', err);
      } finally {
        this.loading = false;
      };
}}}
</script>

<style scoped>
.phone-numbers-container {
  padding: 20px;
}

.data-container {
  padding: 20px;
  max-height: 70vh;
  overflow-y: auto;
}

.data-container::-webkit-scrollbar {
  width: 8px;
}

.data-container::-webkit-scrollbar-track {
  background: rgba(255, 255, 255, 0.2);
  border-radius: 4px;
}

.data-container::-webkit-scrollbar-thumb {
  background: linear-gradient(135deg, #1890ff, #096dd9);
  border-radius: 4px;
}

.card {
  background: white;
  border-radius: 20px;
  margin-bottom: 24px;
  overflow: hidden;
  position: relative;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.12);
}

.card::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  height: 4px;
  background: linear-gradient(90deg, #1890ff, #69c0ff, #1890ff);
}

.card-header {
  background: white;
  padding: 20px 24px;
  border-bottom: 1px solid #f0f0f0;
}

.card-header h3 {
  margin: 0;
  color: #1890ff;
  font-size: 18px;
  font-weight: 700;
  background: linear-gradient(135deg, #1890ff, #096dd9);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.card-body {
  padding: 20px 24px;
  background: white;
}

.info-row {
  display: flex;
  padding: 12px 0;
  border-bottom: 1px solid #f5f5f5;
}

.info-row:last-child {
  border-bottom: none;
}

.label {
  font-weight: 600;
  width: 200px;
  flex-shrink: 0;
  color: #8c8c8c;
  font-size: 13px;
  letter-spacing: 0.3px;
}

.value {
  flex: 1;
  color: #262626;
  font-size: 14px;
  font-weight: 500;
  word-break: break-word;
}

.card {
  animation: fadeInUp 0.4s ease-out;
}

.numbers-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.search-section {
  display: flex;
  gap: 10px;
}

.phone-input {
  padding: 8px 12px;
  border: 1px solid #ddd;
  border-radius: 4px;
  width: 200px;
}

.search-btn {
  padding: 8px 16px;
  background: #2196f3;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.dialog-content {
  margin-top: 20px;
  padding: 20px;
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

.vtable {
  display: flex;
  flex-direction: column;
}

.table-header {
  background: #167ccf;
  color: white;
  padding: 10px;
  border-radius: 4px;
  margin: 10px 0;
}

.table-row {
  display: flex;
  padding: 8px 0;
  border-bottom: 1px solid #eee;
}

.request-info-label {
  width: 250px;
  font-weight: 500;
}

.request-info-value {
  flex: 1;
}

.loading {
  text-align: center;
  padding: 20px;
  color: #666;
}

.error-message {
  background: #ffebee;
  color: #d32f2f;
  padding: 12px;
  border-radius: 4px;
  margin-bottom: 20px;
}
</style>