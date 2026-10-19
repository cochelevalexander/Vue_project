<template> <!-- Компонент со всем основным функционалом. Звонки, сохранение статусов и т.д.-->
  <div class="container">
    <My-windowCall class="callwindow"
      :connectionStatusMessage="callStatus"
      :username="username"
      :call="call"
      :pause="pause"
      :micro="micro"
      :callNumber="this.callData.number"
      :timer="formattedTime"
    />

    <div class = "button-autocall">
      <div class = "status">
        <div class="status-selector">
          <label class="custom-label">Проект:</label>
          <select v-model="callData.project">
            <option v-for="(name, key) in selectingProject" :key="key" :value="name">
              {{ name }}
            </option>
          </select>
          <label class="custom-label">Статус звонка:</label>
          <select v-model="callData.status">
            <option v-if="this.callData.project !== 'КАСКО2'" value="Абонент бросил трубку">Абонент бросил трубку</option>
            <option v-if="this.callData.project !== 'КАСКО2'" value="Автоответчик/Секретарь">Автоответчик/Секретарь</option>
            <option v-if="this.callData.project !== 'КАСКО2'" value="Клиент Интач">Клиент Интач</option>
            <option v-if="this.callData.project !== 'КАСКО2'" value="Нет ипотеки">Нет ипотеки</option>
            <option v-if="this.callData.project !== 'КАСКО2'" value="Отказ">Отказ</option>
            <option v-if="this.callData.project !== 'КАСКО2'" value="Отказ от диалога">Отказ от диалога</option>
            <option value="Перезвон">Перезвон</option>
            <option value="Не отвечает">Не отвечает</option>
            <option v-if="this.callData.project !== 'КАСКО2'" value="Черный список">Черный список</option>
            <option v-if="this.callData.project !== 'КАСКО2'" value="Не наш регион">Не наш регион</option>
            <option v-if="this.callData.project !== 'КАСКО2'" value="Переведен на расчет (лид)">Переведен на расчет (лид)</option>
            <option v-if="this.callData.project !== 'КАСКО2'" value="Уже получил расчет">Уже получил расчет</option>
            <option v-if="this.callData.project !== 'КАСКО2'" value="Отказ компании">Отказ компании</option>
            <option v-if="this.callData.project !== 'КАСКО2'" value="Уже продлил">Уже продлил</option>
            <option v-if="this.callData.project !== 'КАСКО2'" value="УТестовый звонок">Тестовый звонок</option>
          </select>
          <VueCtkDateTimePicker v-if="callData.status==='Перезвон'" v-model="callData.callbackTime" label="Дата перезвона" format="YYYY-MM-DD HH:mm:ss" minute-interval="5"/>
          <button @click="submitData" class="submit-btn" id="submitBtn">
            <span class="btn-text">Отправить</span>
          </button>
          

          
        </div>
        
      </div>
      <div :class="{ 'has-button': callData.project !== 'КАСКО' }" class="open-number">
        <OpenNumber :project="this.callData.project" :number="this.callData.number" @request-updated="handleRequestData"/>
      </div>
      <div v-if="!callTrasfer && this.callData.project == 'КАСКО'" class="incom"><button class="button-incoming" id ="holdon" @click="startTransferLoop"><img src="free-icon-forward-1373451.png" class="icon-small"></button></div>
      <div v-if="callTrasfer && this.callData.project == 'КАСКО'" class="incom"><button class="button-incoming" id ="holdon" @click="stopTransferLoop"><img src="free-icon-call-17043235.png" class="icon-small"></button></div>
      <div v-if="!pause"  class="incom"><button class="button-incoming" id ="holdon" @click="holdCall"><img src="32x32-3706883-control-music-pause-stop-stop-music_108702.png"></button></div>
      <div v-if="pause" class="incom"><button class="button-incoming" id ="unholdon" @click="unholdCall"><img src="free-icon-call-168.png"></button></div>
      <div v-if="micro" class="incom"><button class="button-incoming" id ="microphone" @click="offmicro"><img src="3643759-mic-microphone-recorder-speak.png"></button></div>
      <div v-else class="incom"><button class="button-incoming" id ="unmicrophone" @click="onmicro"><img src="iconfinder-mute-mic-microphone.png"></button></div>
      <div class="incom"><button class="button-incoming" id ="complete" @click="declineCall"><img src="free-icon-font-phone-call.png"></button></div>
    </div>
    <div>
    </div>
    <div class = "window-arrangement">
        <!--<My-SendMessageYandex v-if="buttonStates[0]" />-->
        <Customer_card :request="this.displayRequestData" :project="this.callData.project" :number="this.callData.number" :flag = this.flag  class= "client-card"/>
        <My-messangeMax v-if="buttonStates[0] "  class="img-icon-fixed"/>
        <My-SendMessageVk v-if="buttonStates[0] && this.flag" />
        <My-phone v-if="buttonStates[1]" @call="makeCall" @decline-call="declineCall" @transfer-call="transfer"  v-model="callData.number" class ="fone" />
        <My-notebook v-if="buttonStates[2]" />
    </div>
  </div>
</template>

<script>
import JsSIP from 'jssip';
import { markRaw } from 'vue';
import { mapState, mapActions } from 'vuex'
import { mapGetters} from 'vuex';
import VueCtkDateTimePicker from 'vue-ctk-date-time-picker';
import 'vue-ctk-date-time-picker/dist/vue-ctk-date-time-picker.css';
import axios from 'axios';
export default {

  data() {
    const projects = this.$store.getters.getUserProjects || {};
    const firstProject = Object.values(projects)[0] || '';
    return {
      isTransferring: false,
      displayRequestData: null,
      canCompleteTransfer: false,
      flag: false,
      transferStatus: '',
      originalCallSession: null,
      queueCallSession: null,
      queueTimeout: null,
      transfercall: false,
      ua: null,
      callTrasfer: false,
      callStartTime: null,
      callSession: null,
      callId: null,
      callStatus: 'ожидание',
      callNumber: null,
      showModal: false,
      timeStartCall: null,
      timeEndCall: null, 
      incomingCallNumber: null,
      controllCall: false,
      dial: true, 
      call: true,
      pause: false,
      micro: true,
      ID: null,
      incoming: false,

      selectingProject: projects,
      project: null,
      time: null,
      timerInterval: null,
      seconds: 0,
      callData: {
        number: null,
        startTime: null,
        endTime: null,
        user: this.$store.getters.getCredentials.username,
        status: null,  
        comment: null,
        durations: null,
        success: null,
        record: false,
        callbackTime: null,
        provider: null,
        project: firstProject
      }
    };
  },
  computed: {
    ...mapGetters(['isAuthenticated','selectedStatus','buttonStates', 'call', 'timeElapsed']),
    formattedTime() {
      const minutes = Math.floor(this.seconds / 60);
      const secs = this.seconds % 60;
      return `${minutes}:${secs < 10 ? '0' : ''}${secs}`;
    },
    ...mapState(['phoneWindow']), 
   
  },
  components:{
    VueCtkDateTimePicker
  },
  watch: {
    selectedStatus(newValue) {
    this.dialup(newValue); 
  },
    phoneWindow(newValue) {
      this.phone = newValue; 
    },
    call(newVal) {
      this.toggleCall(newVal)
    },
    isAuthenticated(newVal){
      const state = 'inactive'
      this.dialup(state)
    }
  },
  methods: {
    ...mapActions(['updateCall']),

    formatDate(date = new Date()){
          const year = String(date.getFullYear())
          const month = String(date.getMonth() + 1).padStart(2, '0')
          const day = String(date.getDate()).padStart(2, '0')
          const hours = String(date.getHours()).padStart(2, '0')
          const minutes = String(date.getMinutes()).padStart(2, '0')
          const secondes = String(date.getSeconds()).padStart(2, '0')
          return `${year}-${month}-${day} ${hours}:${minutes}:${secondes}`;
      },

    playCallAlert(type = 'start') {
    const ctx = new (window.AudioContext || window.webkitAudioContext)();
    const osc = ctx.createOscillator();
    const gain = ctx.createGain();
    
    const settings = {
        start: {type: 'sine', frequency: [349.23, 392.00], duration: 0.8, gain: 0.35, fadeIn: 0.1, fadeOut: 0.4}, //, interval: 0.2
        end: {type: 'triangle', frequency: [800, 400], duration: 0.8, gain: 0.5, fadeIn: 0.1, fadeOut: 0.7}};
    
    const config = settings[type] || settings.start;

    osc.type = config.type;

    if (config.frequency.length > 1) {
        osc.frequency.setValueAtTime(config.frequency[0], ctx.currentTime);
        osc.frequency.exponentialRampToValueAtTime(config.frequency[1], ctx.currentTime + config.duration * 0.5);
        if (config.frequency.length > 2) {
            osc.frequency.exponentialRampToValueAtTime(config.frequency[2], ctx.currentTime + config.duration);
        }
    } else {
        osc.frequency.value = config.frequency[0];
    }

    gain.gain.setValueAtTime(0, ctx.currentTime);
    gain.gain.linearRampToValueAtTime(config.gain, ctx.currentTime + config.fadeIn);
    gain.gain.linearRampToValueAtTime(0, ctx.currentTime + config.fadeOut);

    osc.connect(gain);
    gain.connect(ctx.destination);
    
    osc.start();
    osc.stop(ctx.currentTime + config.duration);

    osc.onended = () => {
        if (ctx.state !== 'closed') {
            ctx.close();
        }
    };
    setTimeout(() => {
        if (ctx.state !== 'closed') {
            ctx.close();
        }
    }, (config.duration + 0.1) * 1000);
    },
    handleBeforeUnload(event) { //Определение закрытия браузера
      event.preventDefault();
      const status = 'inactive'
      this.dialup(status);
    },

    async connectToAsterisk() {
      JsSIP.debug.enable('JsSIP:*');
      var socket = new JsSIP.WebSocketInterface('wss://domain:port/wss')
      var configuration = {
        sockets: [socket],
        uri: `sip:${this.username}@domain`,
        session_timers: false,
      };
      this.ua = markRaw(new JsSIP.UA(configuration));

      this.ua.on('registered', () => {
        this.dialup(this.selectedStatus)
        this.$store.dispatch('updateConnectionStatus', true);
      });

      this.ua.on('registrationFailed', (data) => {
        this.$store.dispatch('updateConnectionStatus', false);
      });
      this.ua.on('message', function(request) {
      });
         
      this.ua.on('newRTCSession', (data) => {
        const session = data.session;
        if (session.direction === 'incoming' || session.direction === 'outgoing') {
          if (!this.callData.number){
            if(session.remote_identity.display_name){
            this.callData.number = session.remote_identity.display_name}
            console.log(this.callData.number)
            this.transfercall = true
          }
          if(!this.callData.number){
              if(session.remote_identity.uri.user){
            this.callData.number = session.remote_identity.uri.user}
          }
          console.log('Номер с автика', this.callData.number)
          this.ID = (performance.now()/1000).toFixed(8)
          this.callData.success = this.callData.number
          if (this.callData.number){
            this.ua.sendMessage(`sip:${this.username}@domain`, this.callData.success);
          }
          this.controllCall = true

          
          console.log(session)
          this.playCallAlert('start');
          this.ua.sendMessage(`sip:${this.username}@domain`, "incall");
          this.handleIncomingCall(session) 
          console.log("Входящий звонок")
          //this.callId = session.id;
          if (session.connection) {
          session.connection.ontrack = (event) => {
            const remoteAudio = document.createElement('audio');
            remoteAudio.srcObject = event.streams[0];
            remoteAudio.play();
            if (!this.timerInterval) {
        this.timerInterval = setInterval(() => {
          this.seconds++;
        }, 1000);
      }
          }};
          session.on('ended', this.handleCallEnd);
          session.on('failed', this.handleCallEnd);
          session.on('accepted', () => {
          });
          
         
        } 
        
      });
      

      this.ua.start();
    }, 
    soundplay(){
        this.playCallAlert('start'); // Добавлено чисто для проверки разных звуков
    },
//{ value: 'active', label: 'Активно' },
//        { value: 'complete', label: 'Неактивен' },
//        { value: 'manual', label: 'Ручной дозвон' },
//        { value: 'pause', label: 'Перерыв' },
//        { value: 'learning', label: 'Обучение' },
//        { value: 'technical', label: 'Тех.пауза' }
    dialup(dial) {
      if (!this.ua) {return;}
      try {
        if (dial === 'active' && this.callData.project !== 'КАСКО2') {
          this.ua.sendMessage(`sip:${this.username}@domain`, "active");
        } else if(dial === 'pause') {
          this.ua.sendMessage(`sip:${this.username}@domain`, "pause");
          //this.ua.stop()
        }else if(dial === 'manual') {
          this.ua.sendMessage(`sip:${this.username}@domain`, "manual");
        }else if(dial === 'active' && this.callData.project === 'КАСКО2') {
          this.ua.sendMessage(`sip:${this.username}@domain`, "active2line");
        }else if(dial === 'inactive') {
          this.ua.sendMessage(`sip:${this.username}@domain`, "inactive");
        }else if (dial === 'learning'){
          this.ua.sendMessage(`sip:${this.username}@domain`, "learning");
        }else if(dial === 'technical'){
          this.ua.sendMessage(`sip:${this.username}@domain`, "technical");
        }
      } catch (error) {
        console.error('Ошибка отправки статуса:', error);
      }
},
    
    toggleCall(newVal) {
      const newCallStatus = this.call;
      this.updateCall(newCallStatus);},   

    handleIncomingCall(session) {
      this.incomingCallNumber = session.remote_identity.uri.toString(); 
      this.callNumber = this.incomingCallNumber.match(/SIP:(\d+)@/);
      this.call = false;
      this.callSession = session;

      this.timeStartCall = new Date()
      this.callData.startTime = this.formatDate(this.timeStartCall)

      if (session.direction === 'incoming') {
          this.incoming = true
          this.call = true 
          console.log("Прилетело с автикка")
          console.log(session)
          if (this.callSession) {
            this.callSession.answer({mediaConstraints: { audio: true, video: false }}); 
            this.callSession.connection.ontrack = (event) => {
                const remoteAudio = document.createElement('audio');
                remoteAudio.srcObject = event.streams[0];
                remoteAudio.play();
                if (!this.timerInterval) {
            this.timerInterval = setInterval(() => {
              this.seconds++;
            }, 1000);}};}
        }
      if (session.direction === 'outgoing') {
          this.incoming = false
        }

      session.on('ended', this.handleCallEnd);
      session.on('failed', this.handleCallEnd);
    },
    declineCall() {
      this.showModal = false;
      this.transfercall = false; 
      this.dialup(this.selectedStatus) 
      if (this.callSession) {
        this.callSession.terminate(); 
        this.callSession = null; 
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
      }
    },
    async submitData() {
      if(this.controllCall){
        if (!this.authToken) {
        await this.getAuthToken();
        if (this.error) return; 
      }

      
      try {
        const { data } = await axios.post('/api/addPost', this.callData, {
          headers: { 
            Authorization: `Bearer ${this.authToken}` 
          },
          timeout: 30000 
        });

        console.log('Данные успешно отправлены:', data);
        this.dialup(this.selectedStatus)
        this.callData.status = null; 
        this.loading = true;
        this.error = null;
        this.loading = false;
        this.callData.number = null
        this.callData.startTime = null
        this.callData.endTime = null
        this.callData.status = null
        this.callData.comment = null
        this.callData.durations = null
        this.callData.success = null
        this.callData.record = null
        this.callData.callbackTime = null
        this.callData.callbackTime = null
        this.callData.number = null
        this.controllCall = false
        alert('Данные отправлены');
        
      } catch (err) {
        this.error = 'Ошибка отправки данных';
        console.error('Submit error:', err);
        alert('Данные отправлены', err);

        if (err.response?.status === 401) {
          this.authToken = null; 
        }
      }
      }
  },


    holdCall() {
      if (this.callSession){
      this.isProcessing = true;
      this.pause = true;
      this.callSession.hold({
        useUpdate: true,
        eventHandlers: {
          succeeded: () => {
            this.isProcessing = false;
            console.log('Звонок поставлен на удержание');
          },
          failed: (e) => {
            console.error('Ошибка при постановке на удержание:', e);
            this.pause = false;
            this.isProcessing = false;
          }
        }
      });}
  },
    unholdCall() {
      if (this.callSession){

        this.isProcessing = true;
        this.pause = false;
        this.callSession.unhold({
          eventHandlers: {
            succeeded: () => {
              this.isProcessing = false;
              console.log('Звонок снят с удержания');
            },
            failed: (e) => {
              this.pause = true;
              console.error('Ошибка при снятии с удержания:', e);
              this.isProcessing = false;
            }
          }
        });}

    },
    onmicro() {
      if (this.callSession){
      this.callSession.unmute()
      this.micro = true}

  },

    offmicro() {
      if (this.callSession){
      this.callSession.mute()
      this.micro = false}
    },
    handleRequestData(payload) { 
      console.log('Получены данные от OpenNumber:', payload)
      this.displayRequestData = payload.request
      this.flag = true
    },

    startTransferLoop() {
      if (this.callSession){
        this.holdCall()
        if (this.isTransferLoopActive) return;
        this.callTrasfer = true
        
        this.isTransferLoopActive = true;
        console.log('Проверка активности');
        this.transfer();
        this.transferLoopInterval = setInterval(() => {
          if (this.isTransferLoopActive) {
            this.transfer();
          }
        }, 1000); // Таймер потом настроить под запрос 
      }
  },

  stopTransferLoop() {
    if (this.callSession){
      this.isTransferLoopActive = false;
      this.callTrasfer = false
      this.unholdCall()
      if (this.transferLoopInterval) {
        clearInterval(this.transferLoopInterval);
        this.transferLoopInterval = null;
      }
      
      console.log('Пауза');
      this.counter = 0;
    }
  },

 
  async transfer() {
    if (!this.isTransferLoopActive) return;  
    this.counter++;
    console.log(`Трансфер #${this.counter}`, new Date().toLocaleTimeString());
    
    try {
      const { data } = await axios.get('/api/kasko2lne', {
        headers: { 
          Authorization: `Bearer ${this.authToken}` 
        },
        timeout: 30000 
      });
      
      console.log(`Ответ #${this.counter}:`, data);
      console.log(data.data)
      if (data.data != null){
        console.log(data.data);
        this.stopTransferLoop();
        await this.callSession.refer('sip:transfer_queue@domain', {
    extraHeaders: [
    `Variable: ${this.callData.number}`,
    ],
    eventHandlers: {
        'referProgress': (data) => {
            console.log('Статус перевода:', data);
        },
        'referAccepted': (data) => {
            console.log('Перевод завершен успешно');
            this.isTransferring = false;
            this.declineCall(); 
        },
        'referFailed': (data) => {
            console.error('Ошибка перевода:', data);
            this.isTransferring = false;
        }
    }
});
        
      }
      return data;
      
    } catch (error) {
      console.error(`Ошибка #${this.counter}:`, error.message);
    }
  },
    makeCall(number) {
      if (!this.callSession && this.callData.project !== null && this.selectedStatus === 'manual') {
          this.pause = false;
          this.call = true;
          this.controllCall = true;
        //this.callData.project = this.project;
        if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia) {
          console.error('getUserMedia не поддерживается в этом браузере.');
          return;
        }
        this.callNumber = number;
        this.callData.number = number;
        navigator.mediaDevices.getUserMedia({ audio: true, video: false })
          .then((stream) => {
            const options = {
              mediaStream: stream,
              mediaConstraints: { audio: true, video: false },
              eventHandlers: {
                'progress': () => { console.log('Звонок в процессе'); },
                'failed': () => {
                  console.log('Звонок не удался');
                  this.callStatus = 'ошибка';
                },
                'confirmed': () => { console.log('Звонок подтвержден'); }
              }
            };
            this.callSession = this.ua.call(number, options);
          })
          .catch((error) => {
            console.error('Ошибка доступа к медиа-устройствам:', error);
          });
      }else if (this.callData.project === null) {
        alert('Выберите проект');}
      },

      handleCallEnd() {
        this.pause = false;
        this.micro = true;
        this.timeEndCall = new Date()
        this.callData.endTime = this.formatDate(this.timeEndCall)
        const duration = Math.round((this.timeEndCall-this.timeStartCall)/1000)
        this.callData.durations = duration
        this.callData.endTime = this.formatDate(this.timeEndCall)
        console.log(this.callData)
        this.callStartTime = null; // Сброс
        this.callSession = null;
        this.callId = null;
        this.callStatus = 'ожидание';
        this.showModal = false;
        clearInterval(this.timerInterval);
        this.timerInterval = null;
        //this.dialup(this.selectedStatus); Отправка о смене статуса
        this.seconds = 0;
        this.playCallAlert('end');
      },
      ...mapActions(['updateButtonState']), 
      setButtonState(index) {
        this.updateButtonState(index); 
      },
    },
    mounted() {
        this.connectToAsterisk();
        window.addEventListener('beforeunload', this.handleBeforeUnload);
      },

  name: 'My_Aster'
};
</script>

<style scoped>
.client-card{
  margin-right: 10%;
}

.client-card.has-button {
  margin-right: clamp(0%, 6vw, 45%);
}
@media (min-width: 1800px) and (max-width: 2700px) {
  .client-card.has-button {
    margin-right: clamp(26.4%, 6vw, 45%);
  }
}
@media (min-width: 1800px) and (max-width: 2700px) {
  .client-card{
    margin-right: clamp(22%, 8vw, 65%);
  }}



.open-number {
  align-content: center;
}
.img-icon-fixed{
  position: fixed;
}
.open-number.has-button {
  margin-right: clamp(0%, 6vw, 45%);
}
@media (min-width: 1800px) and (max-width: 2700px) {
  .open-number.has-button {
    margin-right: clamp(26.4%, 6vw, 45%);
  }
}
@media (min-width: 1800px) and (max-width: 2700px) {
  .open-number{
    margin-right: clamp(22%, 8vw, 65%);
  }}
.modal {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
}

.modal-content {
  background: linear-gradient(180deg, #345b61, #93cfca);
  width: 30%;
  height: 25%;
  color: rgb(157, 233, 223);
  font-size: 120%;
  border-radius: 10px;
  text-align: center;
}

.button-incoming{
  margin-top: 5px;
}
.callwindow {
  overflow: hidden;
  margin: 0;
  margin-top: clamp(1%, 2.5vw, 8%); 
  margin-left: clamp(0%, 6vw, 10%);
  padding: 0;
  padding-left: clamp(1%, 4vw, 7%);
}
.incom{
  padding: 10px;
  margin: 0;
  display: inline;

}
.window-arrangement {
  display: flex; 
  align-items: flex-start; 
  justify-content: flex-end; 
  height: 100%; 
  gap: 20px; 
  padding-right: 6%;
  min-height: 70vh
}
.button-autocall{
  display: flex;       
  position: relative;
  justify-content: flex-end; 
  margin-right: clamp(3%, 5vw, 10%);

}
#answe{
  background-color: rgb(70, 141, 91);
}
#holdon, #unholdon{
  background-color: rgb(67, 127, 238);
}
.status-selector {
  position: relative;
  margin: 20px;
  padding: 15px;
  border: 1px solid #eee;
  border-radius: 30px;
  display: flex;
  align-items: baseline; 
  gap: 10px;
  margin-left: 20%;
}

select {
  padding: 8px;
  margin-top: 5px;
  width: 100%;
  border: 1px solid #ccc;
  border-radius: 8px;
  cursor: pointer;
}
#microphone, #unmicrophone{
  background-color: rgb(60, 117, 240);
}
#complete{
  background-color: rgb(129, 25, 25);
}
#answe, #complete, #holdon, #microphone, #unmicrophone, #unholdon{
  width: 60px; 
  height: 60px; 
  border: none; 
  border-radius: 50%; 
  color: white; 
  font-size: 16px; 
  cursor: pointer;
  transition: background-color 0.3s; 
}
#answe:hover {
    background-color: rgb(37, 196, 16); 
}

#complete:hover {
    background-color: rgb(212, 17, 17); 
}
#holdon:hover,  #microphone:hover, #unmicrophone:hover, #unholdon:hover{
    background-color: rgb(9, 119, 170); 
}
.status {
  position: relative;
  display: flex;
  flex-direction: row;
  flex-wrap: wrap;

  @media (min-width: 2000px) {
    margin-right: clamp(20%, 25vw, 39%);
  }
  @media (max-width: 1280px) {
    margin-right: clamp(2%, 5vw, 10%);
  }

}
.custom-label {
    color: #ffffff; 
}
.submit-btn {
    position: relative;
    width: 80%;
    background: linear-gradient(135deg, #3498db, #2980b9);
    color: white;
    border-radius: 6px;
    padding: 8px 12px;
    cursor: pointer;
    transition: all 0.3s ease;
    box-shadow: 0 2px 8px rgba(52, 152, 219, 0.3);
    overflow: hidden;
}

.submit-btn:hover:not(.loading) {
    background: linear-gradient(135deg, #2980b9, #3498db);
    transform: translateY(-1px);
    box-shadow: 0 4px 12px rgba(52, 152, 219, 0.4);
}

.submit-btn:active:not(.loading) {
    transform: translateY(0);
}
.icon-small {
  width: 50px;
  height: 50px;
  vertical-align: middle;
  opacity: 0.8;
}

.submit-btn .btn-text {
    transition: opacity 0.3s ease;
}



</style>