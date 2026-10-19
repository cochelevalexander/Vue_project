<template> <!--Отображение параметров: Имя оператора, время разговора, номер телефона, статус подключения к астеру-->
  <div class="callwindow">
    <div class="client-info">
      <div class="info">
        <div class="info-item">
          <p>Статус подключения: {{ connectionStatusMessage }}</p>
        </div>
        <div class="info-item">
          <p>Оператор: {{ username }} {{ currentRole }}</p>
        </div>
        <div class="info-item">
          <p>Номер абонента: {{ callNumber }}</p>
        </div>
        <div class="info-item">
          <p>Время разговора: {{ timer }}</p>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { mapGetters } from 'vuex';

export default {
  props: {
    connectionStatusMessage: String,
    username: String,
    callNumber: String,
    call: Boolean,
    timer: String,
  },

  computed: {
    ...mapGetters(['connectionStatus', 'timeElapsed']), 

    connectionStatusMessage() {
      return this.connectionStatus ? 'Подключено' : 'Отключено';
    },
    currentRole() {
      return this.$store.getters.userRole; 
    },

    formattedTimer() {
      const minutes = Math.floor(this.timer / 60);
      const seconds = this.timer % 60;
      return `${minutes} мин ${seconds} сек`;
    },
  },

  name: 'My-windowCall',
};
</script>
  
<style scoped>
.button-incoming {
    margin-top: 5px;
}

.incom {
    padding: 10px;
    margin: 0;
    display: inline;
}

.info {
    color: aliceblue;
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
    gap: 10px;
}

.info-item {
    border: 2px solid aliceblue;
    border-radius: 10px; 
    background-color: rgba(255, 255, 255, 0.1); 
    text-align: center; 
}

.info-item p {
    margin: 0; 
}
</style>