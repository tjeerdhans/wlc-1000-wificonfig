<template>
  <div id="app">    
    <Configurator></Configurator>    
  </div>
</template>

<script>
import Configurator from './components/Configurator.vue'

export default {
  name: 'app',
  components: {
    Configurator
  }
}
</script>

<style>
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  /* text-align: center; */
  color: #2c3e50;
  /* margin-top: 60px; */
}
</style>
