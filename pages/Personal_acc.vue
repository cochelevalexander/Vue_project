<template> <!-- Личный кабинет оператора-->
  <My-Cap-Home class="cap-home" v-if="showOperatorsLink"/>
  <div class="container">
    <div class="arrange-statistics">
      <div class="inline-fields-container">
        <div class = "datatime">
              <VueCtkDateTimePicker v-model="dataCall.startTime"  label="Время начала" format="YYYY-MM-DD HH:mm:ss" minute-interval="5"/>
              <VueCtkDateTimePicker v-model="dataCall.endTime"  label="Время завершения" format="YYYY-MM-DD HH:mm:ss" minute-interval="5"/>
              <button @click="submitData" class="submit-btn" id="submitBtn">
                  <span class="btn-text">Получить</span>
                </button>
        </div>  
        <div class="stats-header">
          <div class="stat-card white-card">
            <div class="stat-icon"><img src="free-icon-pie-chart-3650590.png" class="icon-small"></div>
            <div class="stat-info">
              <div class="stat-label">Статистика оператора</div>
              <div class="stat-name">{{ dataCall.Myname }}</div>
            </div>
          </div>
        </div>

        <div class="stats-overview">
          <div class="stat-card white-card">
            <div class="stat-icon"><img src="free-icon-call-7721646.png" class="icon-small"></div>
            <div class="stat-info">
              <div class="field-label">Количество звонков</div>
              <div class="field-value">{{ countCallsByOperator(dataCall.Myname) }}</div>
            </div>
          </div>
      
          <div class="stat-card white-card" id="time">
            <div class="stat-icon"><img src="free-icon-time-5058458.png" class="icon-small"></div>
            <div class="stat-info">
              <div class="field-label">Общее время разговора</div>
              <div class="field-value">{{ formatDuration(getTotalDurationByOperator(dataCall.Myname)) }}</div>
            </div>
          </div>
        </div>
        <el-slider v-model="volume" :min="0" :max="100"></el-slider>
        
      </div>
      <div v-if="callsList && callsList.length > 0" class="status-call-section">
          <h3 class="section-title">Распределение статусов звонков</h3>
          <div class="status-grid">
            <div v-for="stat in getStatusPercentages()" :key="stat.status" class="status-item white-card">
              <div class="status-header">
                {{ stat.status }}: {{ stat.percentage }}%
              </div>
            </div>
          </div>
        </div>

      <div class="chart-container-large">
          <Pie id="my-chart-id" :options="chartOptions" :data="chartData" />
      </div>
    </div>
  </div>
</template>

<script>
import { mapGetters, mapActions } from 'vuex';
import axios from 'axios';
import VueCtkDateTimePicker from 'vue-ctk-date-time-picker';
import 'vue-ctk-date-time-picker/dist/vue-ctk-date-time-picker.css';
import { Pie, Bar, Line } from 'vue-chartjs'
import { Chart as ChartJS, ArcElement, Title, Tooltip, Legend, BarElement, CategoryScale, LinearScale } from 'chart.js'
ChartJS.register(Title, Tooltip, Legend, BarElement, CategoryScale, LinearScale, ArcElement)

export default {
  data() {
    return {
    callsList: null,

    dataCall: {
        startTime: null,
        endTime: null, 
        Myname: this.$store.getters.getCredentials.username,

      },
      chartData: {
        labels: [], 
        datasets: [{ 
          data: [], //,'5','6','7','8','9','10','11','12','13',
          backgroundColor: ['#FF6B6B', '#FF8E53', '#FFD166', '#A8E6CF', '#6A9FB5',
  '#7B68EE', '#DDA0DD', '#9ACD32', '#FF6347', '#FFA500',
  '#FFD700', '#32CD32', '#1E90FF', '#9370DB'],
          borderColor: ['#ffffff'],
 
        }],
        
      },
      chartOptions: {
        responsive: true,
        maintainAspectRatio: false,
        plugins: {
          title: {
            display: true,
            text: 'Статистика звонков',
            color: '#030201'
          }
        },
        scales: {
        y: {
          beginAtZero: true
        }
      },
      },

    };
  },
  computed: {
    ...mapGetters(['userRole', 'isAdmin']), 
        showOperatorsLink() {
            return ['manager','distributor', 'admin'].includes(this.userRole);
        },
  },
  components:{
    VueCtkDateTimePicker,
    Bar,
    Pie
  },
  watch: {
    callsList: {
      handler(newCalls) {
        if (newCalls && newCalls.length > 0) {
          this.updateChartData(newCalls);
        }
      },
      deep: true
    }
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
    getStatusPercentages() {
      const total = this.callsList.length;
      const counts = {};

      this.callsList.forEach(call => {
        const status = call.статус || 'Без статуса';
        counts[status] = (counts[status] || 0) + 1;
      });
        console.log(status)
        const allStatuses = Object.entries(counts).map(([status, count]) => ({
        status,
        percentage: ((count / total) * 100).toFixed(1)
      }));

      console.log('Все статусы:', allStatuses);
      return allStatuses
    },
    updateChartData(calls) {
      const counts = {};
      calls.forEach(call => {
        const status = call.статус || 'Без статуса';
        counts[status] = (counts[status] || 0) + 1;
      });
      
      const labels = Object.keys(counts); 
      const data = Object.values(counts); 
      
      const total = calls.length;
      const percentages = data.map(count => 
        parseFloat(((count / total) * 100).toFixed(1)) 
      );
      
      this.chartData = {
        labels: labels,
        datasets: [{
          data: data, 

        }]
      };
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
        endDate: this.dataCall.endTime,
        operator: this.dataCall.Myname,
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
countCallsByOperator(operator) {
        if (!this.callsList || !this.callsList.length) return 0;

        return this.callsList.length;
    },

getTotalDurationByOperator(operator) {
      if (!this.callsList || !this.callsList.length) return 0;
      
      return this.callsList
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
onCallsDataReceived(data) {

  console.log('Получены данные звонков:', data);
  if(data==0){
    alert('Данные отсутствуют');
  }
},
  },


  name: 'Personal_acc',
};
</script>
  
<style scoped>
.container {
  min-height: 100vh;
  height: auto;
  padding: 20px;
  background: linear-gradient(180deg, #689ce0, #1c3df557);
  /*background: 
        url('Снежинка.png'),
        linear-gradient(180deg, #689ce0, #1c3df557);
        */
  display: flex;
  flex-direction: column;
  align-items: flex-start; 
}
.datatime{
  display: flex;
  padding-bottom: 10px;
  /*margin: 0 auto;*/

}
.icon-small {
  width: 70px;
  height: 70px;
  vertical-align: middle;
  opacity: 0.8;
}

.date-time-picker{
  padding-left: 5px;
}
.statistic{
  color: white;
  size: 3px;
}
.submit-btn{
  background: linear-gradient(180deg, #2e6bbb, #26327557);
  cursor: pointer;
  transition: background-color 0.3s;
  color: white;
  border: 0px;
  margin-left: 10px;
  border-radius: 5px;

}
.submit-btn:hover{
  background-color:rgb(79, 192, 173);

}
.inline-fields-container{
  color: white;
}
.chart-container-large {
  width: 80vw;     
  height: 70vh;     
  max-width: 1200px; 
  max-height: 800px; 
  background-color: #fff;
  border-radius: 20px;
}

.inline-fields-container {
  font-family: 'Segoe UI', Tahoma, Verdana, sans-serif;
  background: transparent;
  padding: 20px;
  max-width: 900px;
  /*margin: 0 auto;*/
}

.stats-header {
  margin-bottom: 25px;
}

.stats-header .stat-card {
  padding: 20px;
  border-radius: 12px;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.05);
  background: white;
  border: 1px solid #e8e8e8;
}

.stat-name {
  font-size: 18px;
  font-weight: 600;
}

.stats-overview {
  gap: 20px;
  margin-bottom: 30px;
  flex-wrap: wrap;
  margin-top: 30px;
}

.stat-card {
  min-width: 200px;
  padding: 20px;
  border-radius: 12px;
  display: flex;
  gap: 15px;
  transition: transform 0.3s ease, box-shadow 0.3s ease;
  background: white;
}

.stat-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
}

.stat-icon {
  font-size: 28px;
  width: 50px;
  height: 50px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: linear-gradient(135deg, #f8f9fa, #e9ecef);
  border-radius: 10px;
}

.stat-info {
  flex: 1;
  color:  black;
}

.field-label {
  font-size: 14px;
  margin-bottom: 5px;
  font-weight: 500;
}

.field-value {
  font-size: 24px;
  font-weight: 700;
  color:  black;
}

.status-call-section {
  margin-top: 20px;
}

.section-title {
  font-size: 16px;
  font-weight: 600;
  color:  white;
  margin-bottom: 20px;
  padding-bottom: 12px;
  border-bottom: 2px solid #f0f0f0;
}

.status-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  gap: 15px;
  font-size: 18px;
  font-weight: 600;
  align-items: center;
}

.status-item {
  padding: 18px;
  border-radius: 10px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);
  transition: all 0.3s ease;
  background: white;
  border: 1px solid #f5f5f5;
}

.status-item:hover {
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
  transform: translateY(-2px);
}

.status-header {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 12px;
  color: #000;
}

.status-icon {
  font-size: 20px;
  width: 36px;
  height: 36px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 8px;
}

.status-name {
  font-size: 14px;
  font-weight: 500;
  color:  black;
  flex: 1;
}

.status-percentage {
  font-size: 24px;
  font-weight: 700;
  color:  black;
  margin-bottom: 8px;
  text-align: center;
}

.status-bar {
  height: 6px;
  background: #f0f0f0;
  border-radius: 3px;
  overflow: hidden;
}

.status-bar-fill {
  height: 100%;
  border-radius: 3px;
  transition: width 1s ease-in-out;
}
.arrange-statistics{
  display: flex;
  gap: 10px; 
  justify-content: space-between; 
}
#time{
  margin-top: 30px;
}
</style>