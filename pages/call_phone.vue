<template> <!-- Страница для отображения компонентов-->
    
    <div class = "window">
        <My-Cap-Home class="cap-home" v-if="showOperatorsLink"/>
        
        <div class="content">
            
            <My_Aster :username="login"/>
        </div>
        <My-Vertical @phone ="setPhone"/>
</div>


</template>
        
<script>
import { mapGetters, mapActions } from 'vuex';
        
export default {
    data() {
        return {
            isChecked: true
        };
    },
    computed: {
    ...mapGetters(['buttonStates']), 
    ...mapGetters(['userRole', 'isAdmin']), 
        showOperatorsLink() {
            return ['manager','distributor', 'admin'].includes(this.userRole);
        },
  },
    methods: {
      ...mapActions(['setphoneActive']), 
      ...mapActions(['logout']),
          handleLogout() {
              this.logout().then(() => {
                  this.$router.push('/entrance'); 
              }).catch((error) => {
                  console.error("Выход не удался:", error);
              });
        },
  
      setPhone() {
        const newValue = !this.$store.state.phoneWindow; 
        this.setphoneActive(newValue); 
      },
    },
} 
</script>
    
<style scoped>
.window {
    height: 100%;
    width: 100%;
    margin: 0;
    background: linear-gradient(180deg, #689ce0, #1c3df557);
    min-height: 100vh;
    display: flex;
    flex-direction: column;
    position: relative;
}

.content {
    margin-left: 0;
    padding: 0;
    padding-top: 60px; 
    flex-grow: 1;
    background: linear-gradient(180deg, #387eda, #89a5f357);
    width: 100%;
    position: relative;
    z-index: 1;
}

.cap-home {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    z-index: 10;
    background: inherit; 
}


@media (max-width: 768px) {
    .content {
        padding-top: 50px; 
    }
    
    .cap-home {
        height: 50px;
    }
}

@media (max-width: 480px) {
    .content {
        padding-top: 45px;
    }
    
    .cap-home {
        height: 45px;
    }
}

.translated-ltr {
    height: 100%;
}
</style>