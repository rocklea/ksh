<template>
  <div id="app">
    <ChinaMap />
  </div>
</template>

<script>
import ChinaMap from './components/ChinaMap.vue'

export default {
  name: 'App',
  components: {
    ChinaMap
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
}
</style>
