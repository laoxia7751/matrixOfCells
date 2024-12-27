<template>
  <div class="wrapper">
    <div class="canvasBox">
      <Grid :mockData="mockData"/>
    </div>
  </div>
</template>

<script setup>
import { mockData } from './components/mock';
import Grid from './components/Grid.vue';
</script>

<style scoped>
.canvasBox{
  width: 100%;
  height: 100%;
}
.wrapper{
  width: 100%;
  height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
  background: #000;
}
</style>
