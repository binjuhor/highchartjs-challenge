<template>
  <div class="flex">
    <div class="chartLeft"><ScatterChart :chart-id="1"/></div>
    <div class="chartRight"><ScatterChart :chart-id="2"/></div>
  </div>
</template>

<script setup>
  import { ref, provide } from 'vue';
  import ScatterChart from '@/components/ScatterChart.vue'

  const hoveredPointIndex = ref(null);
  const selectedPointIndex = ref(null);
  provide('hoveredPointIndex', hoveredPointIndex);
  provide('selectedPointIndex', selectedPointIndex);
</script>

<style scoped lang="scss">
  .flex {
    display: flex;
    gap: 30px;
    justify-content: center;
    height: 100vh;
    padding-top: 5vw;

    div {
      max-width: 600px;
    }
  }
</style>
