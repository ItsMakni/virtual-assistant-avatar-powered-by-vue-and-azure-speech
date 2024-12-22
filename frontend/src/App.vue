<!-- App.vue -->
<template>
  <div id="app" :class="currentTheme">
    <!-- <header>
      <Navbar />
    </header> -->
    <main>
      <Chatbot />
      <!-- <Avatar /> -->
    </main>
  </div>
</template>

<script>
import Navbar from './components/Navbar.vue';
import Avatar from './components/Avatar.vue';
import Chatbot from './components/ChatBot.vue';


export default {
  name: 'App',
  components: {
    Chatbot,
    Navbar,
    Avatar,
  },
  data() {
    return {
      currentTheme: 'light',
    };
  },
};
</script>

<style>
/* Your existing styles, including the ones you provided */
.light-mode {
  background-image: url("@/assets/Screenshot.png");
  background-size: cover;
  background-repeat: no-repeat;
  height: 300px;
  width: 300px;}


</style>