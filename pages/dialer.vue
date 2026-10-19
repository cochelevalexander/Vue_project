<template> <!-- Тут для автодозвона. Передача номера телефона из эксельки и тд.-->
  <My-Cap-Home class="cap-home"/>
  <div class="container">
    <div 
      @dragover.prevent="onDragOver"
      @drop.prevent="onDrop"
      @dragleave.prevent="onDragLeave"
      @click="$refs.file.click()"
      class="drop-zone"
      :class="{ 'drag-over': isDragOver }"
    >
      Перетащите Excel файл сюда или кликните
      <input
        ref="file"
        type="file"
        accept=".xlsx,.xls,.csv"
        @change="onChange"
        hidden
      >
    </div>

    <div class="common-fields">
      <h3>Общие параметры:</h3>
      <div class="fields-grid">
        <div class="field">
          <label class="custom-label">Проект:</label>
          <select v-model="common.project">
            <option v-for="(name, key) in selectingProject" :key="key" :value="name">
              {{ name }}
            </option>
          </select>
        </div>
        <div class="field">
          <label>Интенсив (0.2 - 2):</label>
          <div class="slider-container">
            <input 
              type="range" 
              v-model="common.intensive"
              min="0.2" 
              max="2" 
              step="0.1"
              class="slider"
              @input="updateSliderValue"
            />
            <span class="slider-value">{{ common.intensive }}</span>
          </div>
        </div>
      </div>
    </div>

    <div v-if="uploadedFile" class="file-info">
      <div class="file-details">
        <span class="file-name">{{ uploadedFile.name }}</span>
        <span class="file-size">{{ formatFileSize(uploadedFile.size) }}</span>
        <span class="file-count" v-if="phones.length > 0">
          {{ phones.length }} номеров
        </span>
        <button @click="removeFile" class="remove-btn">×</button>
      </div>
      

    </div>

    <button 
      @click="sendData" 
      class="send-btn"
    >
      Отправить
    </button>
  </div>
</template>

<script>
import * as XLSX from 'xlsx';
import axios from 'axios';
export default {
  name: 'My-dialer',
  
  data() {
    return {
      isDragOver: false,
      selectingProject: this.$store.getters.getUserProjects,
      uploadedFile: null,
      phones: [],
      common: {
        project: '',
        intensive: 1.0 
      }
    };
  },
  
  methods: {
    onDragOver() {
      this.isDragOver = true;
    },
    
    onDragLeave() {
      this.isDragOver = false;
    },
    
    onDrop(e) {
      this.isDragOver = false;
      const file = e.dataTransfer.files[0];
      this.processFile(file);
    },
    
    onChange(e) {
      const file = e.target.files[0];
      this.processFile(file);
    },
    
    async processFile(file) {
      if (!file) return;
      
      // Проверка типа файла
      const validTypes = [
        'application/vnd.ms-excel',
        'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
        'text/csv'
      ];
      
      if (!validTypes.includes(file.type) && 
          !file.name.match(/\.(xlsx|xls|csv)$/i)) {
        alert('Пожалуйста, загрузите файл');
        return;
      }
      
      this.uploadedFile = file;
      
      try {
        const phones = await this.parseExcelFile(file);
        this.phones = phones;

        if (!this.common.project) {
          const fileName = file.name.replace(/\.[^/.]+$/, ""); 
          this.common.project = fileName;
        }
        
      } catch (error) {
        console.error('Ошибка при чтении файла:', error);
        alert('Ошибка при чтении файла. Проверьте формат.');
      }
    },
    
    parseExcelFile(file) {
      return new Promise((resolve, reject) => {
        const reader = new FileReader();
        
        reader.onload = (e) => {
          try {
            const data = XLSX.read(e.target.result, { type: 'binary' });
            const sheetName = data.SheetNames[0];
            const sheet = data.Sheets[sheetName];

            const jsonData = XLSX.utils.sheet_to_json(sheet, { header: 1 });
 
            const phones = [];
            jsonData.forEach(row => {
              if (Array.isArray(row)) {
                row.forEach(cell => {
                  if (cell && typeof cell === 'string') {
                    const phone = this.extractPhone(cell);
                    if (phone) {
                      phones.push(phone);
                    }
                  }
                });
              } else if (typeof row === 'object') {
                Object.values(row).forEach(value => {
                  if (value && typeof value === 'string') {
                    const phone = this.extractPhone(value);
                    if (phone) {
                      phones.push(phone);
                    }
                  }
                });
              }
            });
            
            resolve(phones);
          } catch (error) {
            reject(error);
          }
        };
        
        reader.onerror = () => reject(new Error('Ошибка чтения файла'));
        reader.readAsBinaryString(file);
      });
    },
    
    extractPhone(text) {

      const phonePatterns = [

        /\+7\s?\(?(\d{3})\)?\s?(\d{3})[- ]?(\d{2})[- ]?(\d{2})/,
        /8\s?\(?(\d{3})\)?\s?(\d{3})[- ]?(\d{2})[- ]?(\d{2})/,
        /7\s?\(?(\d{3})\)?\s?(\d{3})[- ]?(\d{2})[- ]?(\d{2})/,
        /(\d{3})[- ]?(\d{3})[- ]?(\d{2})[- ]?(\d{2})/,
        /(\d{11})/
      ];
      
      const str = text.toString();
      
      for (const pattern of phonePatterns) {
        const match = str.match(pattern);
        if (match) {
          let phoneDigits;
          
          if (pattern.toString().includes('11')) {
            phoneDigits = match[0];
          } else {
            phoneDigits = match.slice(1).join('');
          }
          
          phoneDigits = phoneDigits.replace(/\D/g, '');

          if (phoneDigits.length === 11) {
            if (phoneDigits.startsWith('7')) {
              return phoneDigits;
            } else if (phoneDigits.startsWith('8')) {
              return '7' + phoneDigits.substring(1);
            }
          } else if (phoneDigits.length === 10) {
            return '7' + phoneDigits;
          }
        }
      }
      
      return null;
    },
    
    async sendData() {
      if (!this.common.project) {
        alert('Заполните поле "Проект"');
        return;
      }
      
      if (!this.uploadedFile) {
        alert('Загрузите Excel файл');
        return;
      }
      
      if (this.phones.length === 0) {
        alert('В файле не найдены телефонные номера');
        return;
      }
      if (!this.authToken) {
        await this.getAuthToken();
        if (this.error) return; 
      }
      try {
        const resultJson = {
        common: {
          project: this.common.project,
          intensive: parseFloat(this.common.intensive), 
          importDate: new Date().toISOString(),
          totalPhones: this.phones.length,
          fileName: this.uploadedFile.name
        },
        phones: this.phones
      };
      console.log('ОТПРАВКА ДАННЫХ:');
      console.log(JSON.stringify(resultJson, null, 2));
      
      alert(`Данные успешно обработаны!\n\n Файл: ${this.uploadedFile.name}\nНомеров: ${this.phones.length}\n Интенсив: ${this.common.intensive}\n`);
        const { data } = await axios.post('/api/startAutodial', resultJson, {
          headers: { 
            Authorization: `Bearer ${this.authToken}` 
          },
          timeout: 30000 
        });

        console.log('Данные успешно отправлены:', data);
       
      } catch (err) {
        this.error = 'Ошибка отправки данных';
        console.error('Submit error:', err);

        if (err.response?.status === 401) {
          this.authToken = null; 
        }
      }
    },

    removeFile() {
      this.uploadedFile = null;
      this.phones = [];
      this.$refs.file.value = '';
    },
    async getAuthToken() {
      try {
        const { data } = await axios.post('/api/login', {
          login: this.$store.getters.getCredentials.username,
          password: this.$store.getters.getCredentials.password
        });
        this.authToken = data.token;
      } catch (err) {
        this.error = 'Ошибка авторизации';
        console.error('Auth error:', err);
      }
    },
    
    updateSliderValue(e) {
      this.common.intensive = parseFloat(e.target.value).toFixed(1);
    },
    
    formatFileSize(bytes) {
      if (bytes === 0) return '0 Bytes';
      const k = 1024;
      const sizes = ['Bytes', 'KB', 'MB'];
      const i = Math.floor(Math.log(bytes) / Math.log(k));
      return parseFloat((bytes / Math.pow(k, i)).toFixed(1)) + ' ' + sizes[i];
    }
  }
};
</script>

<style scoped>
.container {
  min-height: 100vh;
  height: auto;
  padding: 20px;
  background: linear-gradient(180deg, #689ce0, #1c3df557);
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: 20px;
}

.drop-zone {
  border: 2px dashed #ccc;
  padding: 40px;
  text-align: center;
  cursor: pointer;
  border-radius: 8px;
  width: 100%;
  max-width: 600px;
  background: white;
  transition: all 0.3s ease;
  font-size: 16px;
}

.drop-zone.drag-over {
  border-color: #4CAF50;
  background-color: rgba(76, 175, 80, 0.1);
}

.drop-zone:hover {
  border-color: #666;
}

.common-fields {
  background: white;
  padding: 20px;
  border-radius: 8px;
  width: 100%;
  max-width: 600px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

.common-fields h3 {
  margin: 0 0 20px 0;
  color: #333;
  font-size: 18px;
}

.fields-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 20px;
}

.field {
  display: flex;
  flex-direction: column;
}

.field label {
  margin-bottom: 8px;
  font-weight: 600;
  font-size: 14px;
}

.field input[type="text"] {
  padding: 10px 12px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 14px;
  transition: border-color 0.3s;
}

.field input[type="text"]:focus {
  outline: none;
  border-color: #2055c5;
  box-shadow: 0 0 0 2px rgba(76, 175, 80, 0.1);
}

.slider-container {
  display: flex;
  align-items: center;
  gap: 15px;
}

.slider {
  flex: 1;
  height: 6px;
  -webkit-appearance: none;
  appearance: none;
  background: linear-gradient(to right, #ddd, #172dac);
  border-radius: 3px;
  outline: none;
}

.slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #1c2ec9;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 2px 4px rgba(0,0,0,0.2);
}

.slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #234acc;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 2px 4px rgba(0,0,0,0.2);
}

.slider-value {
  min-width: 40px;
  text-align: center;
  font-weight: bold;
  color: #000000;
  font-size: 16px;
}

.file-info {
  background: linear-gradient(180deg, #0ad43c57, #ffffff);
  padding: 15px;
  border-radius: 8px;
  width: 100%;
  max-width: 600px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

.file-details {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 15px;
  margin-bottom: 10px;
}

.file-name {
  font-weight: bold;
  color: #333;
  flex: 1;
  text-align: left;
  font-size: 14px;
}

.file-size, .file-count {
  color: #666;
  font-size: 13px;
  white-space: nowrap;
}

.remove-btn {
  background: none;
  border: none;
  color: #ff4444;
  font-size: 24px;
  cursor: pointer;
  padding: 0 10px;
  line-height: 1;
  transition: color 0.3s;
}

.remove-btn:hover {
  color: #cc0000;
}

.phone-preview {
  margin-top: 10px;
  padding-top: 10px;
  border-top: 1px solid #eee;
}

.phone-preview p {
  margin: 0 0 8px 0;
  color: #666;
  font-size: 14px;
}

.phone-list {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}

.phone-tag {
  background: #e3f2fd;
  padding: 4px 10px;
  border-radius: 15px;
  font-size: 12px;
  color: #1976d2;
  font-family: monospace;
}

.phone-more {
  color: #666;
  font-size: 12px;
  align-self: center;
}

.send-btn {
  background: linear-gradient(180deg, #2e6bbb, #26327557);
  color: white;
  border: none;
  padding: 15px 40px;
  border-radius: 8px;
  font-size: 18px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
  margin-top: 10px;
  box-shadow: 0 4px 6px rgba(0,0,0,0.1);
}

.send-btn:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 6px 12px rgba(46, 88, 179, 0.3);
  background: linear-gradient(135deg, #26327557, #2e6bbb);
}

.send-btn:active:not(:disabled) {
  transform: translateY(0);
}

.send-btn:disabled {
  background: #cccccc;
  cursor: not-allowed;
  opacity: 0.7;
  box-shadow: none;
}
@media (max-width: 768px) {
  .fields-grid {
    grid-template-columns: 1fr;
    gap: 15px;
  }
  
  .drop-zone {
    padding: 30px 20px;
    font-size: 14px;
  }
  
  .send-btn {
    padding: 12px 30px;
    font-size: 16px;
  }
  
  .file-details {
    flex-wrap: wrap;
  }
}
</style>