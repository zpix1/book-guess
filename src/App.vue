<template>
  <div id="app">
    <h1>Угадай книгу по началу</h1>
    <MainView/>
  </div>
</template>

<script>
import MainView from './components/MainView.vue'

export default {
  name: 'App',
  components: {
    MainView
  }
}
</script>

<style>
@import url('https://fonts.googleapis.com/css2?family=Roboto&family=Roboto+Slab&display=swap');
#app {
  text-align: center;
  font-family: 'Roboto', sans-serif;
}

h1, h2, h3 {
  margin-top: 0;
  margin-bottom: 15px;
}
</style>
