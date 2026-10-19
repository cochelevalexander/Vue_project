<template><!-- Отображение телефона для набора номера-->
  <div class="phone" @keydown="handleKeyPress" tabindex="0">
    <div class="display-container">
      <input 
        type="text" 
        class="display-input" 
        v-model="displayValue" 
        @input="validateInput"
        @paste="handlePaste"
        placeholder="Введите номер"
        ref="displayInput"
        maxlength="11"
      />
    </div>

    <div class="buttons">
      <button class="button del" @click="deleteLastDigit">Del</button>
      <button class="button call" @click="makeCall" :disabled="!isValidNumber"><img src="free-icon-font-phone-call.png"></button>
      <button class="button end" @click="transfer">r</button>
      <button v-for="n in [1, 2, 3, 4, 5, 6, 7, 8, 9, 0]"
              :key="n"
              class="button"
              @click="appendNumber(n)"
              :disabled="displayValue.length >= 11">{{ n }}</button>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      displayValue: '',
      isValidNumber: false,
      sip: null,
      callSession: null,
      showModal: false,
    };
  },
  methods: {
    appendNumber(n) {
      if (this.displayValue.length < 11) {
        this.displayValue += n.toString();
        this.validateInput();
      }
    },
    deleteLastDigit() {
      this.displayValue = this.displayValue.slice(0, -11);
      this.validateInput();
    },
    validateInput() {
      const phoneRegex = /^79\d{9}$/;
      this.isValidNumber = phoneRegex.test(this.displayValue);
      
      if (this.displayValue.length > 11) {
        this.displayValue = this.displayValue.slice(0, 11);
      }

      this.displayValue = this.displayValue.replace(/\D/g, '');
    },
    handlePaste(event) {
      event.preventDefault();
      const pastedText = (event.clipboardData || window.clipboardData).getData('text');
      const numbersOnly = pastedText.replace(/\D/g, '').slice(0, 11);
      
      if (numbersOnly.startsWith('79')) {
        this.displayValue = numbersOnly;
      } else if (numbersOnly.length > 0) {
        const remainingDigits = 11 - 2; 
        this.displayValue = '79' + numbersOnly.slice(0, remainingDigits);
      }
      
      this.validateInput();
    },
    makeCall() {
      this.$emit('call', this.displayValue);
    },
    answerCall() {
      this.$emit('answer-call', this.displayValue); 
    },
    declineCall() {
      this.$emit('decline-call'); 
    },
    transfer() {
      this.$emit('transfer-call'); 
    },
    holdCall() {
      this.$emit('hold-call'); 
    },
    showIncomingCall() {
      this.showModal = true; 
    },
    handleKeyPress(event) {
      const key = event.key;

      if (!isNaN(key) && key !== ' ' && this.displayValue.length < 11) {
        this.appendNumber(key);
        event.preventDefault();
      } else if (key === 'Backspace') {
        this.deleteLastDigit();
        event.preventDefault();
      } else if (key === 'Enter' && this.isValidNumber) {
        this.makeCall();
        event.preventDefault();
      } else if ((event.ctrlKey || event.metaKey) && key === 'v') {
      } else if ((event.ctrlKey || event.metaKey) && key === 'м') {
      } else if ((event.ctrlKey || event.metaKey) && key === 'М') {
      } else if ((event.ctrlKey || event.metaKey) && key === 'V') {
      } else {
        event.preventDefault();
      }
    }
  },
  watch: {
    displayValue(newValue) {
      if (newValue.length === 1 && newValue !== '7') {
        this.displayValue = '79' + newValue;
      } else if (newValue.length === 1 && newValue === '7') {
      } else if (newValue.length === 2 && newValue.startsWith('7') && newValue !== '79') {
        this.displayValue = '79' + newValue.charAt(1);
      }
      
      this.validateInput();
    }
  },
  name: 'My-phone'
};
</script>

<style scoped>

.phone {
  background-color: rgba(32, 136, 184, 0.9); 
  border-radius: 10px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
  width: 17em;
  padding: 20px;
  text-align: center;

  margin-left: 7%;
}
.display-input {
  background-color: #cbd0d1;
  color: #000000;
  padding: 15px;
  font-size: 1.2rem;
  border-radius: 5px;
  border: none;
  width: 100%;
  text-align: right;
  box-sizing: border-box;
}

.display-input:focus {
  outline: 2px solid #4CAF50;
}
.display-container {
  background-color: #cbd0d1;
  color: #000000;
  font-size: 1.2rem;
  border-radius: 5px;
  margin-bottom: 15px;
  text-align: right;
}

.buttons {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 10px;
  justify-items: center;
  align-items: center;   
}

.button {
  background-color: #e0e0e0;
  border: none;
  border-radius: 50%; 
  padding: 15px;
  font-size: 1.2rem;
  cursor: pointer;
  transition: background-color 0.3s;
  width: 60px; 
  height: 60px; 
}

.button.del {
  background-color: rgb(238, 238, 125); 
}

.button.call {
  background-color: rgb(102, 219, 102); 
}

.button.end {
  background-color: rgb(224, 91, 91); 
}

.button:active {
  background-color: #ccc;
}
.button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.button.call:disabled {
  background-color: #ccc;
}

.display-input:invalid {
  border: 2px solid red;
}

.display-input:valid {
  border: 2px solid green;
}
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
  background-color: #fff;
  padding: 20px;
  border-radius: 10px;
  text-align: center;
}
</style>
