c:<template>
    <div id = "hed"> <!-- Шапка основной страницы-->
        <div class = "table-string">
            <router-link to=""><img src="logo-Mix.png" id="logo-Mix" alt="лого"></router-link>
        </div>
        <div class="span">
            <div class = "table-string">
                <router-link to="/test" class="cap">
                        <span class="main-links">
                        Отчёты
                        </span>
                </router-link>
            </div>
            <div class = "table-string">
                <router-link to="/phone" class="cap">
                        <span class="main-links">
                        Телефон
                        </span>
                </router-link>
            </div>
            <div class = "table-string">
                <router-link to="/analytics" class="cap">
                        <span class="main-links">
                        Аналитика
                        </span>
                </router-link>
            </div>
            <div class="table-string" >
            <router-link to="/operators" class="cap">
                <span class="main-links">
                    Операторы
                </span>
            </router-link>
        </div>

            <div class="table-string" id="entrance">
                <a @click.prevent="handleLogout" class="cap">
                    <span class="main-links">Выход</span>
                </a>
            </div>
        </div>

</div>
</template>
    
<script>
import { mapActions, mapGetters } from 'vuex';

export default {
    name: 'My-Cap-Home',
    computed: {
        ...mapGetters(['userRole', 'isAdmin']),
        showOperatorsLink() {
            return ['manager', 'admin'].includes(this.userRole);
        },
    },
    methods: {
        ...mapActions(['logout']),
        handleLogout() {
            this.logout().then(() => {
                this.$router.push('/entrance'); 
            }).catch((error) => {
                console.error("Выход не удался:", error);
            });
        }
    }
};
</script>
    
<style scoped>
#hed {
      display: flex;                    
      justify-content: space-between;   
      align-items: center;               
      padding: 10px;                    
      background: linear-gradient(180deg, #4a86f5, #f5e0ff);
      padding-left:4%;
      bottom:92%;   
}
.span {
      display: flex;                   
      justify-content: center;           
      flex: 1;                           
}

.table-string{
      margin: 0 30px;
      transition: all .6s ease; 

                      
}
.main-links {
    color: rgb(243, 243, 243);
    padding: 0%;
    font-size: 1.7em;
    text-align: center;
    vertical-align: middle;
    display: inline-block;
    white-space: nowrap;
}
#logo-Mix{
    width: 180px;
}

#entrance{
    margin-left: auto;    
    margin-right: 8%;     
}

.table-string:hover, a:hover{
    color: #043b3d;
    text-decoration: none;
    transform: scale(1.1);
    
}
</style>