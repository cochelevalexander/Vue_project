<template>
  <div>
    <My-Cap-Home class="cap-home"/>
    <div class = "layout-row">
      <div class="left-window">
        <div class ='window-button'>
          <button class="add-items" > Словари</button>
          <button class="add-items" > Теги</button>
          <button class="add-items" > Чек листы</button>
        </div>
      </div>
      <div class ="search-parameters">
        <div style="position: relative;">
          <VueCtkDateTimePicker v-model="dataCalltime.startTime"  label="Время начала" format="YYYY-MM-DD HH:mm:ss" :append-to-body="true" minute-interval="5"/>
        </div>
        <div style="position: relative;">
          <VueCtkDateTimePicker v-model="dataCalltime.endTime"  label="Время начала" format="YYYY-MM-DD HH:mm:ss" :append-to-body="true" minute-interval="5"/>
        </div>

          <VueMultiselect v-model="selectedStatus" mode="tags" :options="allOptions" :searchable="true" :create-tag="true" placeholder="Статус" class="multiselect" />
          <input type="text" v-model="filterAgent" placeholder="Оператор" class="normal">
          <input type="text" v-model.number="filterPhone" placeholder="Телефон" class="normal">
           <button class="but" @click="download_json"> Получить</button>
      </div>

      <div v-if="filterRequests.length" class="calls-container">
        <div v-for="call in filterRequests" :key="call.id" 
            class="call-item" @click="openCallDetails(call)">
          <div class="call-info">
            <p><strong>ID:</strong> {{ call.id }}</p>
            <p><strong>Кампания:</strong> {{ call.кампания }}</p>
            <p><strong>Номер:</strong> {{ call.номер }}</p>
            <p><strong>Время начала:</strong> {{ call.время_начала }}</p>
            <p><strong>Статус:</strong> {{ call.статус }}</p>
          
          </div>
        </div>

      </div>

      <div v-if="selectedCall" class="modal" @click.self="closeModal">
        <div class="modal-content">
          <span class="close" @click="closeModal">&times;</span>
          <div class="modal-details">
            <div class = "data-call">
              <p><strong>ID:</strong> {{ selectedCall.id }}</p>
              <p><strong>Кампания:</strong> {{ selectedCall.кампания }}</p>
              <p><strong>Номер:</strong> {{ selectedCall.номер }}</p>
              <p><strong>Оператор:</strong> {{ selectedCall.оператор }}</p>
              <p><strong>Время начала:</strong> {{ selectedCall.время_начала }}</p>
              <p><strong>Длительность:</strong> {{ selectedCall.длительность }} сек</p>
              <p><strong>Статус:</strong> {{ selectedCall.статус }}</p>
            </div>
            <audio ref="audioRef" controls :src="audioUrls[selectedCall.id]"></audio>

            
            <h3>Расшифровка</h3>
              <div v-if="selectedCall.теги">
              <div v-for="(item, idx) in selectedCall.теги" :key="idx" class="transcript-item">
                <div class="transcript-header">
                  <span class="timestamp">[{{ item.start }} - {{ item.end }}]</span>
                  <span :class="item.client ? 'client-label' : 'operator-label'">
                    {{ item.client ? ' Клиент:' : (item.operator ? ' Оператор:' : '') }}
                  </span>
                </div>
                
                <div class="transcript-content">
                  <div class="transcript-text" @click="setTargetTime(item.start)">
                    {{ item.client ? item.client[0] : (item.operator ? item.operator[0] : '') }}
                  </div>

                  <div v-if="item.tags && item.tags.length" class="tags-container">
                    <span v-for="(tag, tagIdx) in item.tags" :key="tagIdx" class="tag">
                      {{ tag }}
                    </span>
                  </div>
                </div>
              </div>
            </div>
              <div v-else>
                <div v-for="(item, idx) in selectedCall.расшифровка" :key="idx" class="transcript-item">
                  <div class="transcript-header">
                    <span class="timestamp">[{{ item.start }} - {{ item.end }}]</span>
                    <span :class="item.client ? 'client-label' : 'operator-label'">
                      {{ item.client ? ' Клиент:' : (item.operator ? ' Оператор:' : '') }}
                    </span>
                  </div>
                  <div class="transcript-text" @click="setTargetTime(item.start)">
                    {{ item.client ? item.client[0] : (item.operator ? item.operator[0] : '') }}
                  </div>
                </div>
              </div>
          </div>
          <div class="raiting">

          </div>
        </div>
        
      </div>
    </div>
    
  </div>
</template>
    
<script>
import axios from 'axios';
import { ref } from 'vue'
import VueCtkDateTimePicker from 'vue-ctk-date-time-picker';
import 'vue-ctk-date-time-picker/dist/vue-ctk-date-time-picker.css';
import VueMultiselect from '@vueform/multiselect'
import '@vueform/multiselect/themes/default.css'
export default {
    data() {
        return {
            callsList: [],
            selectedCall: null,
            audioUrls: {},
            targetTime: null,
            loading: false,
            selectedStatus: ref([]),
            allOptions: ['Перевод с котировкой', 'Отказ', 'Отказ от диалога', 'Переведен на расчет', 'Черный Список', 'Перезвон', 'Котировка_Каско. Недозвон до 2-й линии'],
            operatoName: null,
            filterPhone: '', 
            filterAgent: '',
            filterStatus: '',
            dataCalltime:{
              startTime: null,
              endTime: null,
            }

        };
    },
    components:{
    VueCtkDateTimePicker,
    VueMultiselect,
  },
    methods:{
        parseTimeToSeconds(timeStr) {
          if (!timeStr) return 0;

          if (typeof timeStr === 'number') return timeStr;

          if (timeStr.includes(':')) {
            const parts = timeStr.split(':');
            const minutes = parseInt(parts[0], 10) || 0;
            const seconds = parseInt(parts[1], 10) || 0;
            return minutes * 60 + seconds;
          }

          return parseFloat(timeStr) || 0;
        },
        setTargetTime(time) {         
            const seconds = this.parseTimeToSeconds(time);
            if (seconds >= 0) {
              this.targetTime = seconds;
              console.log(this.targetTime);
            } else {
              console.warn('Некорректное время:', time);
            }
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
            }},
        openCallDetails(call) {
            this.selectedCall = call;
            this.loadAudio(call.id)
            document.body.style.overflow = 'hidden'; 
        },
        closeModal() {
            this.selectedCall = null;
            this.audioUrls = {}
            document.body.style.overflow = ''; 
        },
        async getAudio(id){
          const params = {
              id: id,
            };

          const response = await this.$axios.get('/api/getaudio', {
                params: params,
                responseType: 'blob', 
                headers: {
                'Authorization': `Bearer ${this.authToken}`
                }
            });
          console.log('Обращение ок')
          console.log(response.data)
          console.log(response)
           if (response.data.type === 'application/json') {
            const text = await response.data.text();
            console.error( JSON.parse(text));
            throw new Error(errorData.message || 'Server error');
          }
          const audioUrl = URL.createObjectURL(response.data);
          return audioUrl;
        },
        async loadAudio(id) {
          if (this.audioUrls[id]) return;         
            try {
              const audioUrl = await this.getAudio(id);

              this.audioUrls[id] = audioUrl
              
            } catch (error) {
              console.error('Ошибка:', error);
            }
        },

        async download_json() {
            console.log(this.dataCalltime)
            if (!this.authToken) {
              await this.getAuthToken();
              if (this.error) return;
            }
            const params = {
              startDate: this.dataCalltime.startTime,
              endDate: this.dataCalltime.endTime
            };
            try {
            const response = await this.$axios.get('/api/getCallsAnalysis', {
                params: params,
                headers: {
                'Authorization': `Bearer ${this.authToken}`
                }
            });
            if(response.data.data){
                console.log(response.data.data)
                this.callsList = response.data.data
            }
            
            } catch (err) {
            this.error = 'Ошибка отправки запроса';
            console.error('Submit error:', err);

            if (err.response?.status === 401) {
                this.authToken = null;
            }
            }
        },

    },
    computed:{
        filterRequests(){
            let filtered = this.callsList
            if(this.filterPhone){
                filtered = filtered.filter(call => call.номер && call.номер.includes(this.filterPhone));
            }
            if(this.filterAgent){
                filtered = filtered.filter(call => call.оператор && call.оператор.includes(this.filterAgent));
            }
            if(this.selectedStatus){
                if(this.selectedStatus.length >= 1){
                filtered = filtered.filter(call => call.статус && this.selectedStatus.includes(call.статус));
              }

            }
            return filtered;
        },
    },
    watch: {
      targetTime(newTime) {
        const audio = this.$refs.audioRef;
        if (audio && newTime !== null && newTime !== undefined) {
          audio.currentTime = newTime;
          console.log(audio)
          if (audio.paused) {
            audio.play().catch(err => console.warn('Автовоспроизведение заблокировано'));
          }
        }
      }}
    
}
</script>

<style>
.but{
    height: 5vh;
    width: 10%;
    background: linear-gradient(180deg, #689ce0, #1c3df557);
    margin-left: 1%;
}
.add-items{
  height: 5vh;
  width: 90%;
  margin-left: 5%;
  background: linear-gradient(180deg, #689ce0, #1c3df557);
  margin-top: 20%;

}
.data-call {
  display: grid;
  grid-template-columns: repeat(3, 1fr); 
  padding: 16px;
  background: #f8f9fa;
}
.add-items:hover, .but:hover{
  background-color:rgb(79, 192, 173);

}
audio {
  width: 100%;             
  box-shadow: 0 2px 8px rgba(0,0,0,0.2); 
  background: #f0f0f0;      
  display: block;           
}
.window-button{
  margin-top: 35%;
}
.calls-container {
  width: calc(100% - 10vw);
  margin-left: 8vw;
  margin-top: 20px;
  max-height: 70vh;
  overflow-y: auto;
}
.call-item {
  border: 1px solid #ccc;
  margin: 10px;
  padding: 10px;
  border-radius: 5px;
  display: flex;
  flex-direction: column;
  cursor: pointer; 
  transition: transform 0.2s, box-shadow 0.2s;
  background: linear-gradient(180deg, #cee2fd, #4c63e957);
  height: 8vh;

}
.call-item p {
  margin-right: 20px; 
}
.modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.modal-content {
  background: white;
  padding: 20px;
  border-radius: 10px;
  width: 80%;       
  height: 80%;       
  max-width: 80%;     
  max-height: 80%;    
  overflow-y: auto;     
  position: relative;
  display: flex;
  flex-direction: row;
}
.close {
  position: absolute;
  right: 20px;
  top: 10px;
  font-size: 30px;
  cursor: pointer;
  color: #999;
}

.close:hover {
  color: #333;
}

.modal-details {
  margin-top: 20px;
}

.call-item:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
}
.client-label {
  color: #28a745;  
  font-weight: bold;
}

.operator-label {
  color: #007bff; 
  font-weight: bold;
}
.transcript-text{
    padding: 1.5%;
}

.search-parameters {
  width: calc(100% - 10vw);
  margin-left: 8vw;
  padding: 8px 12px;
  display: flex;
  flex-direction: row;

}

.search-parameters > * {
  flex: 1;  
  min-width: 0;  
}
.call-info {
  display: flex;
  flex-direction: row;
  flex-wrap: wrap;
  gap: 20px;
  align-items: center;
}

.call-info p {
  margin: 0;  
  white-space: nowrap; 
}

.left-window {
  height: 100vh;
  width: 8vw;
  min-width: 100px;
  max-height: 100%;
  position: fixed;
  margin-top: 11.5vh;
  left: 0;
  top: 0;
  background: linear-gradient(0deg, #4a86f5, #f5e0ff);
  display: flex;
  flex-direction: column;

}
.normal{
  max-height: 4.3vh;
}
.layout-row {
  display: flex;
  flex-direction: column;
  width: 100%;
}
.transcript-content {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 20px;
  margin-top: 8px;
  padding: 12px;
  background: #f8f9fa;
  border-radius: 8px;
}

.transcript-text {
  flex: 1;
  font-size: 14px;
  color: #2c3e50;
}

.tags-container {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  justify-content: flex-end;
  max-width: 50%;
}

.tag {
  display: inline-block;
  background: #3498db;
  color: white;
  padding: 6px 14px;
  border-radius: 6px;
  font-size: 12px;
  font-weight: 500;
  letter-spacing: 0.5px;
  box-shadow: 0 1px 3px rgba(0,0,0,0.12);
  border: 1px solid rgba(255,255,255,0.2);
}


</style>