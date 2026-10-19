<template> <!-- Страница для аутенфикации-->
  <My-Cap/>
  <div class="conteiner">
    <img id="geometry-right-login" src="geometry.png" alt="пробная картинка">
    <img id="geomety-left-login" src="geomety-left.png" alt="пробная картинка">
    <div class="window-login">
      <form @submit.prevent="handleLogin">
        <h1>Вход</h1>
        <div class="input-box">
          <input type="text" id="log" v-model="login" class="input-data" placeholder="Логин" required>
          <i class='bx bx-user'></i>
        </div>
        <div class="input-box">
          <input type="password" id="pass" v-model="password" class="input-data" placeholder="Пароль" required>
          <i class='bx bxs-lock'></i>
        </div>
        <button type="submit" id="sub-login" class="btn">Войти</button>
        <div v-if="errorMessage" class="error-message">
          {{ errorMessage }}
        </div>
      </form>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      login: '',
      password: '',
      errorMessage: ''
    }
  },
  methods: {
    async handleLogin() {
  try {
    await this.$store.dispatch('login', {
      login: this.login,
      password: this.password
    })
    

    const redirectPath = this.$route.query.redirect || '/phone'
    this.$router.push(redirectPath)
    
  } catch (error) {
    this.errorMessage = error.response?.data?.error || 'Ошибка авторизации'
  }
}
  },
  name: 'Entrance'
}
</script>
     
<style scoped>
    
    
.window-login {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%,-50%);
    min-width: 350px;
    min-height: 300px;
    background: url('/public/gradient.jpg');
    z-index: 3;
    border-radius: 10%;
    padding: 30px 40px;
    display: flex;
    flex-direction: column;
    justify-content: space-between; 
}
.window-login h1{
    color: #fff;
    text-align: center;
}
    
.window-login .input-box{
    width: 100%;
    height: 50px;
    text-align: center;
}
.window-login .input-data{
    width: 100%;
    height: 70%;
    border: none;
    outline: none;
    border-radius: 35px;
    font-size: 20px;
    position: relative;
}
.window-login input::placeholder{
    padding: 45px;
}
    
.input-box i{
    position: absolute;
    right: 20%;
    transform: translateY(-160%);
}
    
.remember{
    display: flex;
    justify-content: space-between;
    margin: 15px 0 15px;
    color: #fff;
    
}
.remember .checkbox{
    width: 30%;
    
}
.window-login .btn {
    width: 100%;
    height: 35px;
    border: none;
    outline: none;
    border-radius: 35px;
    font-size: 20px;
    font-weight: 600;
    transition: all .6s ease;
    margin-top: 20%; 
}
.window-login p {
    text-align: center;
}
.window-login .remember .save{
    justify-content: space-between;
    font-size: 10px;
    height: 60%;
}
.window-login a {
    color: #fff;
}
.window-login .btn:hover {
    transform: scale(1.03);
}

#geometry-right-login {
    position: absolute;
    top: 0;
    right: 0;
    width: 50%;
    height: 100%;
    object-fit: cover;
    z-index: 1;
}

#geomety-left-login {
    position: absolute;
    top: 0;
    left: 0;
    width: 50%;
    height: 100%;
    object-fit: cover;
    z-index: 1;
}
    </style>