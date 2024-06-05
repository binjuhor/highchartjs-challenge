<template>
  <div ref="chartContainer"></div>
  <div class="pointDetail" v-if="pointDetail">
    {{ pointDetail.name }}  -  {{ `ID: ${pointDetail.details[0]}` }} - <strong>{{ `Batch: ${pointDetail.details[1]}` }} - {{ `Product: ${pointDetail.details[4]}` }}</strong>  - {{ `x: ${pointDetail.x} y: ${pointDetail.y}` }}
  </div>
</template>

<script setup>
import { ref, onMounted, inject, defineProps, watch } from 'vue'
import Highcharts from 'highcharts';
import jsonScores from '@/data/json_scores_grafica.json'
import axios from 'axios'

const props = defineProps({
  chartId: Number,
});

const chartContainer = ref(null);
const jsonData = ref(jsonScores[0]);
const pointDetail = ref(null);
const hoveredPointIndex = inject('hoveredPointIndex');
const selectedPointIndex = inject('selectedPointIndex');

onMounted(async () => {
  const objectSpectroscopy = await axios.get('/data/objeto_espectros.json');

  const seriesData = jsonScores.map(serie => ({
    name: serie.name,
    data: serie.data.map(point => ({
      x: point.x,
      y: point.y,
      color: point.color,
      name: point.fullname,
      details: objectSpectroscopy.data.find(item => parseInt(item[12]) === point.fullidA ? item[1] : null),
    }))
  }));

  Highcharts.chart(chartContainer.value, {
    chart: {
      type: 'scatter',
      zoomType: 'xy',
      load: function () {
        this.hoveredPoint = null;
        this.selectedPoint = null;
      },
    },
    title: {
      text: jsonData.value.name,
      align: 'left'
    },
    xAxis: {
      title: {
        enabled: true,
        text: jsonData.value.xAxis
      },
      startOnTick: true,
      endOnTick: true,
      showLastLabel: true
    },
    series: seriesData,
    tooltip: {
      enabled: false,
    },
    plotOptions:{
      series:{
        allowPointSelect: true,
        point:{
          events:{
            mouseOver: (e) => {
              pointDetail.value = e.target
              hoveredPointIndex.value = e.target.index
            },
            mouseOut: () => {
              hoveredPointIndex.value = null
            },
            select: function(event) {
              selectedPointIndex.value = event.target.index;
              this.series.chart.selectedPoint = this;
              this.series.chart.redraw();
            },
            unselect: function() {
              selectedPointIndex.value = null;
              this.series.chart.selectedPoint = null;
              this.series.chart.redraw();
            }
          }
        }
      }
    }
  });
});

watch(hoveredPointIndex, (newIndex) => {
  const chart = Highcharts.charts[props.chartId - 1]
  if (newIndex !== null) {
    if (chart.hoveredPoint) {
      chart.hoveredPoint.setState('');
    }

    chart.hoveredPoint = chart.series[0].data[newIndex];
    chart.hoveredPoint.setState('hover');
    pointDetail.value = chart.hoveredPoint;
  } else {
    pointDetail.value = null;

    if (chart.hoveredPoint) {
      chart.hoveredPoint.setState('');
    }
  }
});

watch(selectedPointIndex, (newIndex) => {
  const chart = Highcharts.charts[props.chartId - 1]
  if (newIndex !== null) {
    const point = chart.series[0].data[newIndex];
    if (point !== chart.selectedPoint) {
      point.select(true, true);
    }
  } else {
    if (chart.selectedPoint) {
      chart.selectedPoint.select(false, true);
    }
  }
  chart.redraw();
});
</script>

<style lang="scss" scoped>
  .pointDetail {
    margin-top: 20px;
    max-width: 70%;
    line-height: 1.5;
    font-weight: 500;

    &::before {
      content: "";
      width: 8px;
      height: 8px;
      border-radius: 20px;
      margin-bottom: 2px;
      background: #00e272;
      display: inline-block;
      margin-right: 5px;
    }
  }
</style>