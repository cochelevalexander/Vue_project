<template> <!-- Отображение активности операторов-->
  <My-Cap-Home />
  <div class="container">
    <div class="data-card">
      <div class="header">
        <h2 class="title">Состояние операторов</h2>
      </div>
      <div class = "datatime">
        <div style="position: relative;"><VueCtkDateTimePicker v-model="dataCall.startTime"  label="Время начала" format="YYYY-MM-DD HH:mm:ss" minute-interval="5"/></div>
        <div style="position: relative;"><VueCtkDateTimePicker v-model="dataCall.endTime"  label="Время завершения" format="YYYY-MM-DD HH:mm:ss" minute-interval="5"/></div>
        <button @click="submitData" class="submit-btn" id="submitBtn">
            <span class="btn-text">Получить</span>
          </button>
      </div>

      <div v-if="loading" class="loader">
        <div class="spinner"></div>
        <span>Загрузка...</span>
      </div>
      
      <div v-else-if="error" class="error-message">
        <span>Ошибка получения данных {{ error }} </span>
        <button @click="fetchData" class="retry-btn">Повторить</button>
      </div>
      
      <div v-else>
        <div v-if="!accounts.length" class="empty-state">
          Нет данных о статусах
        </div>
        
        <div v-else class="accounts-list">
          <div v-for="(item, index) in accounts" :key="index" class="account-item" @click="toggleShowDetails(item.key)">
            <div class="account-field account-main">
              <span class="field-label">Аккаунт: </span>
              <span class="field-value">{{ item.key }}</span>
            </div>
    
    
            <div class="inline-fields-container">
              <div class="inline-field">
                <span class="field-label">Статус:</span>
                <span :class="['status', item.value]">{{ translateStatus(item.value) }}</span>
              </div>
        
              <div class="inline-field" >
                <span class="field-label">Количество звонков:</span>
                <span class="field-value">{{ countCallsByOperator(item.key) }}</span>
              </div>
              <div class="inline-field">
                <span class="field-label">Общее время разговора:</span>
                <span class="field-value">{{ formatDuration(getTotalDurationByOperator(item.key)) }}</span>
              </div>
            </div>
            <div v-if="showDetails[item.key] && getCallsByOperator(item.key).length > 0">
              <table class="calls-table">
                <thead>
                  <tr>
                    <th>ID</th>
                    <th>Проект</th>
                    <th>Номер</th>
                    <th>Время звонка</th>
                    <th>Длительность</th>
                    <th>Статус</th>
                    <th></th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-for="call in getCallsByOperator(item.key)" :key="call.id">
                    <td>{{ call.id }}</td>
                    <td>{{ call.кампания }}</td>
                    <td>{{ call.номер }}</td>
                    <td>{{ call.время_начала }}</td>
                    <td>{{ call.длительность }} сек</td>
                    <td>{{ call.статус }}</td>
                    <td>
                      <a :href="url + call.id" class="download-btn">Скачать</a>
                    </td>
                  </tr>
                </tbody>
              </table>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import axios from 'axios';
import VueCtkDateTimePicker from 'vue-ctk-date-time-picker';
import 'vue-ctk-date-time-picker/dist/vue-ctk-date-time-picker.css';

export default {
  data() {
    return {
      accounts: [],
      openedOperators: new Set(), // Какие операторы открыты
      scrollPosition: 0,
      loading: false,
      error: null,
      authToken: null,
      refreshInterval: null,
      showDetails: {},
      showModal: false,
      callsList: null,
      dataCall: {
        startTime: null,
        endTime: null,

      }
    };
  },
  async created() {
    await this.getAuthToken();
    this.fetchData();
    this.setupAutoRefresh();
    this.submitData();
  },
  beforeUnmount() {
    this.clearAutoRefresh();
  },
  components:{
    VueCtkDateTimePicker
  },
  methods: {
    translateStatus(status) {
      const statusMap = {
        'active': 'Активен',
        'active2line': 'Акт',
        'inactive': 'Неактивен',
        'call': 'В звонке',
        'manual': 'Ручной дозвон',
        'pause': 'Пауза',
        'default': status
      };
      return statusMap[status] || statusMap['default'];
    },
    toggleShowDetails(name){
  if (this.showDetails[name] === undefined || this.showDetails[name] === false  ) {
    this.showDetails[name] = true;;
  }
  else{
    this.showDetails[name] = false;
  }
},
    countCallsByOperator(operator) {
        if (!this.callsList || !this.callsList.length) return 0;

        return this.callsList.filter(call => call.оператор === operator).length;
    },
    getCallsByOperator(operator) {
        if (!this.callsList || !this.callsList.length) return [];
        
        return this.callsList.filter(call => call.оператор === operator);
    },

    getTotalDurationByOperator(operator) {
      if (!this.callsList || !this.callsList.length) return 0;
      
      return this.callsList
        .filter(call => call.оператор === operator)
        .reduce((total, call) => {
          const duration = parseInt(call.длительность) || parseInt(call.duration) || 0;
          return total + duration;
        }, 0);},

    formatDuration(seconds) {
      if (!seconds || seconds === 0) return '00:00:00';
      
      const hours = Math.floor(seconds / 3600);
      const minutes = Math.floor((seconds % 3600) / 60);
      const secs = seconds % 60;
      const pad = (num) => num.toString().padStart(2, '0');
      
      return `${pad(hours)}:${pad(minutes)}:${pad(secs)}`;
},

    
    
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


    async submitData() {
    if (!this.dataCall.startTime || !this.dataCall.endTime) {
      const today = new Date().toISOString().slice(0, 10);
      this.dataCall.startTime = today;
      this.dataCall.endTime = today;
    }

    if (!this.authToken) {
      await this.getAuthToken();
      if (this.error) return;
    }

    try {
      const params = {
        startDate: this.dataCall.startTime,
        endDate: this.dataCall.endTime
      };

      const response = await this.$axios.get('/api/getCalls', {
        params: params,
        headers: {
          'Authorization': `Bearer ${this.authToken}`
        }
      });

      if (response.data.success) {
        this.callsList = response.data.data;

        this.onCallsDataReceived(response.data.data);
      } else {
        this.error = 'Ошибка при получении данных';
      }
      
    } catch (err) {
      this.error = 'Ошибка отправки запроса';
      console.error('Submit error:', err);

      if (err.response?.status === 401) {
        this.authToken = null;
      }
    }
},


onCallsDataReceived(data) {

  console.log('Получены данные звонков:', data);
},
    
    async fetchData() {
  const previouslyOpened = new Set(this.openedOperators);
  const scrollPos = window.scrollY;
  
  try {
    const { data } = await axios.get('/api/kafka/asterisk-accounts', {
      headers: { Authorization: `Bearer ${this.authToken}` }
    });
    
    const newAccounts = data.data?.[0] 
      ? Object.values(data.data[0]).filter(item => 
          item && typeof item === 'object' && item.key
        )
      : [];

    this.accounts = newAccounts.map(account => {
      return {
        ...account,
        isOpened: previouslyOpened.has(account.key)
      };
    });

    this.openedOperators = previouslyOpened;
    
  } finally {
    this.loading = false;
    
    this.$nextTick(() => {
      window.scrollTo(0, scrollPos);
    });
  }
},
    
    setupAutoRefresh() {
      this.refreshInterval = setInterval(() => {
        if (!this.loading) this.fetchData();
      }, 15000);
    },
    
    clearAutoRefresh() {
      if (this.refreshInterval) clearInterval(this.refreshInterval);
    }
  }
};
</script>

<style scoped>
.calls-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 14px;
  background: white;
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.calls-table thead {
  background: linear-gradient(180deg, #689ce0, #1c3df557);
  color: white;
}

.calls-table th {
  padding: 12px 15px;
  text-align: left;
  font-weight: 600;
  font-size: 14px;
}

.calls-table tbody tr {
  border-bottom: 1px solid #eaeaea;
  transition: background 0.2s;
}

.calls-table tbody tr:hover {
  background: #f8f9fa;
}

.calls-table td { 
  padding: 10px 15px;
  color: #1d5692;
}

.download-btn {
  display: inline-block;
  padding: 5px 12px;
  background: #689ce0;
  color: white;
  border-radius: 4px;
  text-decoration: none;
  font-size: 13px;
  transition: background 0.2s;
}

.download-btn:hover {
  background: #007bff;
  color: white;
  text-decoration: none;
}



.container {
  min-height: 85vh;
  padding: 20px;
  background: linear-gradient(180deg, #689ce0, #1c3df557);
  /*background: 
        url('Снежинка.png'),
        linear-gradient(180deg, #689ce0, #1c3df557);
        */ /* Это на случай, если надо будет вернуть нг тему*/
  display: flex;
  flex-direction: column;
  align-items: flex-start; 
}

.data-card {
  background: rgba(255, 255, 255, 0.9);
  border-radius: 12px;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
  padding: 24px;
  backdrop-filter: blur(5px);
  min-width: 95%;
  
}
.submit-btn{
  background: linear-gradient(180deg, #689ce0, #1c3df557);
  cursor: pointer;
  transition: background-color 0.3s;
  color: white;
  border: 0px;
  margin-left: 10px;

}
.submit-btn:hover{
  background-color:rgb(79, 192, 173);

}
.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 24px;
  padding-bottom: 16px;
  border-bottom: 1px solid #e0e0e0;
}

.title {
  margin: 0;
  font-size: 1.5rem;
  color: rgb(28, 99, 158);
  font-weight: 600;
  display: inline-block;
}

.refresh-btn {
  background: #689ce0;
  border: none;
  width: 40px;
  height: 40px;
  border-radius: 50%;
  cursor: pointer;
  color: white;
  font-size: 1rem;
  transition: all 0.3s;
  display: flex;
  align-items: center;
  justify-content: center;
}

.refresh-btn:hover {
  background: #4a87c7;
  transform: rotate(180deg);
}

.loader {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 12px;
  color: #2c3e50;
}

.spinner {
  width: 24px;
  height: 24px;
  border: 3px solid rgba(0, 0, 0, 0.1);
  border-radius: 50%;
  border-top-color: #689ce0;
  animation: spin 1s ease-in-out infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.error-message {
  color: #e74c3c;
  background: #fef0f0;
  padding: 16px;
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 16px;
}

.retry-btn {
  background: #e74c3c;
  color: white;
  border: none;
  border-radius: 6px;
  padding: 8px 16px;
  cursor: pointer;
  font-weight: 500;
  transition: background 0.2s;
}

.retry-btn:hover {
  background: #c0392b;
}
.datatime{
  display: flex;
  width: 35%;
  padding: 10px;

}
.empty-state {
  color: #7f8c8d;
  text-align: center;
  padding: 24px;
  font-size: 1.1rem;
}

.accounts-list {
  display: flex;
  flex-direction: column;
  gap: 16px;
  max-height: 600px;
  overflow-y: auto;
  padding-right: 8px;
}

.accounts-list::-webkit-scrollbar {
  width: 6px;
}

.accounts-list::-webkit-scrollbar-track {
  background: #f1f1f1;
  border-radius: 3px;
}

.accounts-list::-webkit-scrollbar-thumb {
  background: #c1c1c1;
  border-radius: 3px;
}

.accounts-list::-webkit-scrollbar-thumb:hover {
  background: #a8a8a8;
}


.account-item {
  padding: 12px 16px;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  background: white;
}

.account-field {
  display: flex;
  margin-bottom: 8px;
}

.account-field:last-child {
  margin-bottom: 0;
}

.field-label {
  font-weight: 600;
  color: #5a6a7a;
  min-width: 80px;
}

.field-value {
  cursor: pointer;
  color: #2d3748;
  font-size: 13px;
  margin-left: 5px;
}
.status {
  font-weight: 600;
  padding: 2px 8px;
  border-radius: 4px;
}

.status.active {
  color: #27ae60;
  background-color: rgba(39, 174, 96, 0.1);
}
.status.manual {
  color: #abd34e;
  background-color: rgba(39, 174, 96, 0.1);
}
.status.pause {
  color: #f0611e;
  background-color: rgba(39, 174, 96, 0.1);
}
.status.inactive {
  color: #e74c3c;
  background-color: rgba(231, 76, 60, 0.1);
}

.status.call {
  color: #3498db;
  background-color: rgba(52, 152, 219, 0.1);
}

.fa-spin {
  animation: fa-spin 2s infinite linear;
}

@keyframes fa-spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(359deg); }
}

.account-main {
  display: flex;
  margin-bottom: 12px;
  padding-bottom: 8px;
  border-bottom: 1px solid #f1f5f9;
}

.inline-fields-container {
  display: flex;
  align-items: center;
  gap: 24px; 
  flex-wrap: nowrap; 
  overflow-x: auto; 
  padding-bottom: 4px;
}

.inline-field {
  display: flex;
  align-items: center;
  gap: 6px;
  white-space: nowrap; 
  flex-shrink: 0; 
}

.field-label {
  font-weight: 600;
  color: #5a6a7a;
  font-size: 13px;
  min-width: auto; 
}


.status {
  padding: 3px 10px;
  border-radius: 12px;
  font-size: 12px;
  font-weight: 500;
  white-space: nowrap;
}

.status.active { background: #dcfce7; color: #166534; }
.status.pending { background: #fef3c7; color: #92400e; }
.status.inactive { background: #fee2e2; color: #991b1b; }

.actions {
  margin-left: auto; 
  display: flex;
  gap: 8px;
}

.inline-fields-container::-webkit-scrollbar {
  height: 4px;
}

.inline-fields-container::-webkit-scrollbar-track {
  background: #f1f1f1;
  border-radius: 2px;
}

.inline-fields-container::-webkit-scrollbar-thumb {
  background: #c1c1c1;
  border-radius: 2px;
}
</style>