<template> <!-- Элементы боковой панели-->
  <div class="vertical-line">   

            <button class="dropdown_button" 
                @click="toggleList()">
                <div class = "personal"><img src='free-icon-user.png'></img></div>
            </button>

            <div v-if="isDropdownOpen" id="courses_container" class="dropdown-menu">
              <div class="dropdown-item">
                <router-link to="/personal" class="personal-links"><span class="status-label">Личный кабинет</span></router-link>
                <router-link to="/dialer" class="personal-links"  v-if="showOperatorsLink"><span class="status-label">Автодозвон</span></router-link>
                <router-link to="/phone" class="personal-links"><span class="status-label">Настройки</span></router-link>
                <div class="personal-links"><a @click.prevent="handleLogout" class="cap"><span class="status-label">        Выход</span></a></div>
              </div>
            </div>
    <My-switch @status-change="handleStatusChange" />

    <button v-for="(state, index) in buttonStates" 
            :key="index" 
            @click="setButtonState(index)" 
            class="vertical-button">
      <img :src="buttonImages[index]" 
           :alt="'Кнопка ' + (index + 1)" 
           class="button-image" />
    </button>
  </div>
</template>
<script>
import { mapGetters, mapActions } from 'vuex';

export default {
  data() {
    return {
      phone: false,
      toggleState: false,
      isDropdownOpen: false,

      buttonImages: [
        //require('/public/free-icon-broken-14906148.png'), 
        require('/public/free-icon-chat-1041916.png'), 
        require('/public/free-icon-telephone-724664.png'), 
        require('/public/free-icon-address-book-4924134.png'), 
      ],
    };
  },
  computed: {
  ...mapGetters(['buttonStates']), // Состояния кнопок 
  ...mapGetters(['userRole', 'isAdmin']),
  showOperatorsLink() {
            return ['distributor'].includes(this.userRole);
        },
},
  
  methods: {
    ...mapActions(['logout']),
    ...mapActions(['updateButtonState']), 
  setButtonState(index) {
    this.updateButtonState(index); 
  },
  handleLogout() {
            this.logout().then(() => {
                this.$router.push('/entrance'); 
            }).catch((error) => {
                console.error("Выход не удался:", error);
            });
        },
    
  toggleList() {
      this.isDropdownOpen = !this.isDropdownOpen;
    },
  
  },


  name: 'My-Vertical',
};
</script>
  
<style scoped>
.vertical-line {
  position: fixed;
  left: 0;
  top: 0;
  bottom: 0;
  width: auto;
  background: linear-gradient(180deg, #689ce0, #1c3df557);
  border-right: 2px solid black;
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 20px 0;
  z-index: 15;
}

.vertical-button {
  margin: 10px 0;
  padding: 10px 15px;
  cursor: pointer;
  background-color: #8fc7e0;
  color: white;
  border: none;
  border-radius: 8px;
  transition: background-color 0.3s;
}
.dropdown-item {
  padding: 10px 15px;
  cursor: pointer;
  background-color: #b4d6f1;
  justify-content: space-between;
  align-items: center;
}


.vertical-button:hover {
  background-color:rgb(79, 192, 173);
}

.logout-container {
  margin-top: auto;
  padding-bottom: 20px; 
  width: 100%;
  text-align: center;
}
.personal{
  margin: 0;
  padding: 0px 50%;
  cursor: pointer;
  background-color: none;
  border: none;
  border-radius: 8px;
  transition: background-color 0.3s;
  width: 0px;
  
}
.dropdown-menu {
  position: absolute;
  left: 0;
  right: 0;
  background-color: white;
  border: 1px solid #ccc;
  border-radius: 4px;
  margin-top: 55%;
  z-index: 1000;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

.dropdown_button{
  padding: 0px;
  margin-right: 50%;
  font-size: 0px;
  border: none;
  background-color: none;
  margin-bottom: 10%;
}
.main-links {
  color: rgb(177, 200, 218);
  font-size: 1.3em;
  transition: all .6s ease;
  cursor: pointer;
  
}
.status-label{
  color:rgb(3, 67, 70);
  display: inline-block;
  width: 100%;
  margin: 10px 0;
  
}

.main-links:hover {
  color:rgb(131, 225, 228);
  text-decoration: none;
  transform: scale(1.1);
}
.personal-links{
  transition: background-color 0.3s;
  width: 100%;
  

}
.personal-links:hover{
 background-color:rgb(79, 192, 173);
 

}
</style>