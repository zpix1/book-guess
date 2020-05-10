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
#app {
  text-align: center;
}
</style>
