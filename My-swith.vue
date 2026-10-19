<template>
  <div class="status-selector">
    <div class="selected-status" @click="toggleDropdown">
      {{ getStatusLabel(selectedStatus) }}
      <div class="status-timer" >
        {{ formatTime(timers[selectedStatus]) }}
      </div>
    </div>
    
    <div v-if="isDropdownOpen" class="dropdown-menu">
      <div 
        v-for="status in statusOptions" 
        :key="status.value"
        class="dropdown-item"
        :class="{ active: selectedStatus === status.value }"
        @click="selectStatus(status.value)"
      >
        <span class="status-label">{{ status.label }}</span>
      </div>
    </div>
  </div>
</template>

<script>
import { mapActions, mapState } from 'vuex';
import Cookies from 'js-cookie';

export default {
  data() {
    return {
      isDropdownOpen: false,
      statusOptions: [
        { value: 'active', label: 'Активно' },
        { value: 'inactive', label: 'Неактивно' },
        { value: 'manual', label: 'Ручной дозвон' },
        { value: 'pause', label: 'Перерыв' },
        { value: 'learning', label: 'Обучение' },
        { value: 'technical', label: 'Тех.пауза' }
      ],
      timers: {
        active: 0,
        inactive: 0,
        manual: 0,
        pause: 0,
        learning: 0,
        technical: 0
      },
      timerIntervals: {
        active: null,
        inactive: null,
        manual: null,
        pause: null
      },
      saveInterval: null,
      callprocessed: false,
    };
  },

  computed: {
    ...mapState(['selectedStatus', 'timeTalk', 'call']),
    
    selectedStatusLabel() {
      return this.getStatusLabel(this.selectedStatus);
    }
  },

  methods: {
    getStatusLabel(status) {
      const option = this.statusOptions.find(opt => opt.value === status);
      return option ? option.label : 'Неактивно';
    },

    toggleDropdown() {
      this.isDropdownOpen = !this.isDropdownOpen;
    },

    selectStatus(status) {
      if (status === this.selectedStatus) {
        this.isDropdownOpen = false;
        return;
      }

      Cookies.set('selectedStatus', status, { expires: 0.583 });
      
      this.saveTimers();
      
      this.$store.dispatch('updateSelectedStatus', status);
      this.$emit('status-change', status);
      
      if (this.selectedStatus) {
        this.stopTimer(this.selectedStatus);
      }
      this.startTimer(status);
      
      this.isDropdownOpen = false;
    },

    startTimer(status) {
      if (!this.timerIntervals[status]) {
        this.timerIntervals[status] = setInterval(() => {
          this.timers[status] += 1;
          if (this.timers[status] % 10 === 0) {
            this.saveTimers();
          }
        }, 1000);
      }
    },

    stopTimer(status) {
      if (this.timerIntervals[status]) {
        clearInterval(this.timerIntervals[status]);
        this.timerIntervals[status] = null;
      }
    },

    stopAllTimers() {
      Object.keys(this.timerIntervals).forEach(status => {
        this.stopTimer(status);
      });
    },

    saveTimers() {
      Cookies.set('statusTimers', JSON.stringify(this.timers), {
        expires: 0.333,
        secure: true,
        sameSite: 'strict'
      });
    },

    restoreTimers() {
      try {
        const savedTimers = Cookies.get('statusTimers');
        if (savedTimers) {
          const parsedTimers = JSON.parse(savedTimers);
          Object.keys(this.timers).forEach(key => {
            if (parsedTimers[key] !== undefined) {
              this.timers[key] = parsedTimers[key];
            }
          });
        }
      } catch (error) {
        console.error('Ошибка при восстановлении таймеров:', error);
      }
    },

    formatTime(seconds) {
      const hours = Math.floor(seconds / 3600);
      const minutes = Math.floor((seconds % 3600) / 60);
      const secs = seconds % 60;
      
      if (hours > 0) {
        return `${hours}:${minutes.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
      } else {
        return `${minutes}:${secs.toString().padStart(2, '0')}`;
      }
    },

    Call(value) {
      this.callprocessed = value;
    },
  },

  created() {
    this.restoreTimers();
    
    const savedStatus = Cookies.get('selectedStatus');
    if (savedStatus && savedStatus !== this.selectedStatus) {
      this.$store.dispatch('updateSelectedStatus', savedStatus);
    }
    
    if (this.selectedStatus) {
      this.startTimer(this.selectedStatus);
    }
    
    this.saveInterval = setInterval(() => {
      this.saveTimers();
    }, 30000);
  },

  watch: {
    selectedStatus(newStatus, oldStatus) {
      if (oldStatus && oldStatus !== newStatus) {
        this.stopTimer(oldStatus);
      }
      if (newStatus && newStatus !== oldStatus) {
        this.startTimer(newStatus);
      }
    },

    call(newVal) {
      this.Call(newVal);
      if (newVal) {
        this.stopAllTimers();
      } else if (this.selectedStatus) {
        this.startTimer(this.selectedStatus);
      }
    },
  },

  beforeDestroy() {
    this.stopAllTimers();
    clearInterval(this.saveInterval);
    this.saveTimers(); 
  },

  name: 'MySwitch',
};
</script>

<style scoped>
.status-selector {
  position: relative;
  display: inline-block;
  width: 100px;
  font-size: 12px;
}

.selected-status {
  padding: 5px 10px;
  background-color: #0f0a0a;
  color: white;
  border-radius: 34px;
  cursor: pointer;
  display: grid;
  grid-template-columns: 1fr auto 1fr; 
  justify-content: center;
  align-items: center;
  transition: background-color 0.4s;
  position: relative;
  min-height: 40px;
}

.selected-status:hover {
  background-color: #2275d4;
}

.arrow {
  font-size: 12px;
  margin-left: 16px;
}

.status-timer {
  font-size: 10px;
  opacity: 0.8;
  margin-left: 8px;
}

.dropdown-menu {
  position: absolute;
  top: 100%;
  left: 0;
  right: 0;
  background-color: white;
  border: 1px solid #ccc;
  border-radius: 4px;
  margin-top: 5px;
  z-index: 1000;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

.dropdown-item {
  padding: 10px 15px;
  cursor: pointer;
  transition: background-color 0.3s;
  background-color: #b4d6f1;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.dropdown-item:hover {
  background-color: #7db3e6;
}

.dropdown-item.active {
  background-color: #2275d4;
  color: white;
}

.status-label {
  flex: 1;
}

</style>