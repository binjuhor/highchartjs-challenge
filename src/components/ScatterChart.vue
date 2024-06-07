<template>
  <div class="chartWrapper">
    <div ref="chartContainer"></div>
    <div class="contextMenu" v-if="showContextMenu">
      <ul>
        <li :class="selectedIds.length ? 'active' : 'disabled'" @click="handleExclude">
          <img src="/images/upload-big-arrow.png" width="12px" alt="">
          <span>Exclude</span>
        </li>
        <li :class="selectedIds.length ? 'active' : 'disabled'" @click="handleInclude">
          <img src="/images/download.png" width="12px" alt="">
          <span>Include</span>
        </li>
        <li @click="toggleSelectMode">
          <img src="/images/refresh-page-option.png" width="12px" alt="">
          <span>Actual action: "Select Mode {{ selectMode ? 'On' : 'Off'}}"</span>
        </li>
        <li :class="selectedIds.length ? 'active' : 'disabled'">
          <img src="/images/change.png" width="12px" alt="">
          <span>Change selected spectra to class:</span>
          <select v-model="classChangedTo" :disabled="!selectedIds.length">
            <option value="0">Select an option</option>
            <option value="1">Class 1</option>
            <option value="2">Class 2</option>
            <option value="3">Class 3</option>
          </select>
        </li>
        <li @click="handleResetZoom">
          <img src="/images/magnifying-glass.png" width="12px" alt="">
          <span>Reset zoom</span>
        </li>
        <li :class="selectedIds.length ? 'active' : 'disabled'" @click="handleDeletePoints">
          <img src="/images/trash-can.png" width="12px" alt="">
          <span>Delete selected spectra</span>
        </li>
        <li class="border-block" @click="handleRecalculatePCA">
          <img src="/images/refresh-page-option.png" width="12px" alt="">
          <span>Recalculate PCA</span>
        </li>
        <li @click="handleCloseContextMenu">
          <img src="/images/close.png" width="12px" alt="">
          <span>Close</span>
        </li>
      </ul>
    </div>
  </div>
  <div class="pointDetail" v-if="pointDetail">
    {{ pointDetail.name }}  -  {{ `ID: ${pointDetail.details[0]}` }} - <strong>{{ `Batch: ${pointDetail.details[1]}` }} - {{ `Product: ${pointDetail.details[4]}` }}</strong>  - {{ `x: ${pointDetail.x} y: ${pointDetail.y}` }}
  </div>

</template>

<script setup>
import { ref, onMounted, inject, defineProps, watch } from 'vue'
import Highcharts from 'highcharts';
import jsonScores from '@/data/json_scores_grafica.json' // Main data for the chart
import axios from 'axios'

const props = defineProps({
  chartId: Number,
});

const chartContainer = ref(null);
const jsonData = ref(jsonScores[0]);
const pointDetail = ref(null);
const hoveredPointIndex = inject('hoveredPointIndex');
const selectedPointIndex = inject('selectedPointIndex');
const showContextMenu = ref(false  );
const selectedIds = ref([]);
const seriesData = ref([]);
const classChangedTo = ref(0);
const selectMode = ref(false);

const handleGetPointData = async () => {
  try {
    const response = await axios.get('/data/objeto_espectros.json');
    return response.data;
  } catch (error) {
    console.error('Error loading data:', error);
  }
}

const setupChartSeries = async () => {
  try {
    const objectSpectroscopy = await handleGetPointData();

    return jsonScores.map(serie => ({
      name: serie.name,
      data: serie.data.map(point => ({
        id: point.fullidA,
        x: point.x,
        y: point.y,
        color: point.color,
        name: point.fullname,
        originalColor: point.color,
        excluded: false,
        details: objectSpectroscopy.find(item => parseInt(item[12]) === point.fullidA ? item[1] : null),
      }))
    }));
  } catch (error) {
    console.error('Error setting up chart series:', error);
  }
}

onMounted(async () => {
  try{
    seriesData.value = await setupChartSeries();

    Highcharts.chart(chartContainer.value, {
      chart: {
        type: 'scatter',
        zoomType: 'xy',
        load: function () {
          this.hoveredPoint = null;
          this.selectedPoint = null;
        },
        events: {
          click: () => {
            showContextMenu.value = false;
            selectedIds.value = [];
          },
          load:  () => {
            chartContainer.value.addEventListener('contextmenu', (e) => {
              e.preventDefault();
              showContextMenu.value = true;
            });
          }
        }
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
      series: seriesData.value,
      tooltip: {
        enabled: false,
      },
      plotOptions:{
        series:{
          allowPointSelect: selectMode.value,
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
                selectedIds.value.push(event.target.id);
                this.series.chart.selectedPoint = this;
                this.series.chart.redraw();
              },
              unselect: function() {
                selectedIds.value = selectedIds.value.filter(id => id !== this.id);
                selectedPointIndex.value = null;
                this.series.chart.selectedPoint = null;
                this.series.chart.redraw();
              },
            }
          }
        }
      }
    });
  } catch (error) {
    console.error('Error creating chart:', error);
  }
});

// Sync hovered point with hoveredPointIndex
watch(hoveredPointIndex, (newIndex) => {
  const chart = getCurrentChart();

  if (newIndex !== null && chart.series[0].data[newIndex]) {
    if (chart.hoveredPoint && !chart.series[0].data[newIndex] === undefined) {
      chart.hoveredPoint?.setState('');
    }

    chart.hoveredPoint = chart.series[0].data[newIndex];
    chart.hoveredPoint?.setState('hover');
    pointDetail.value = chart.hoveredPoint;
  } else {
    pointDetail.value = null;

    if (chart.hoveredPoint && !chart.hoveredPoint === undefined) {
      chart.hoveredPoint?.setState('');
    }
  }
});

// Sync selected point with selectedPointIndex
watch(selectedPointIndex, (newIndex) => {
  const chart = getCurrentChart();

  if (newIndex !== null && chart.series[0].data[newIndex]) {
    const point = chart.series[0].data[newIndex];
    if (point !== chart.selectedPoint && !point === undefined) {
      point.select(true, true);
    }
  } else {
    if (chart.selectedPoint && !chart.selectedPoint === undefined) {
      chart.selectedPoint.select(false, true);
    }
  }
  chart.redraw();
});

watch(classChangedTo, (newClass) => {
  const chart = getCurrentChart();

  if (selectedIds.value.length === 0 || newClass === 0) {
    return;
  }

  const colors = ['red', 'blue', '#5b5fc6'];

  selectedIds.value.forEach(id => {
    const point = chart.series[0].data.find(point => point.id === id);
    point.update({
      color: colors[newClass - 1],
      selected: false
    });
  });

  resetContextMenu();
});

const getCurrentChart = () => {
  return Highcharts.charts[props.chartId - 1]
}

const resetContextMenu = () => {
  selectedIds.value = [];
  showContextMenu.value = false;
  classChangedTo.value = 0;
}

const handleInclude = () => {
  const chart = getCurrentChart();

  if (selectedIds.value.length === 0) {
    return;
  }

  selectedIds.value.forEach(id => {
    const point = chart.series[0].data.find(point => point.id === id);
    point.update({
      color: point?.originalColor,
      selected: false
    });
  });

  resetContextMenu();
}

const handleExclude = () => {
  const chart = getCurrentChart();

  if (selectedIds.value.length === 0) {
    return;
  }

  selectedIds.value.forEach(id => {
    const point = chart.series[0].data.find(point => point.id === id);
    point.update({
      color: 'rgba(0, 0, 0, 0.1)',
      selected: false,
      excluded: true,
    });
  });

  resetContextMenu()
}

const handleCloseContextMenu = () => {
  showContextMenu.value = false;
}

const toggleSelectMode = () => {
  const chart = getCurrentChart();
  selectMode.value = !selectMode.value;

  chart.series[0].update({
    allowPointSelect: selectMode.value
  });
  resetContextMenu();
}

const handleResetZoom = () => {
  const chart = getCurrentChart();
  chart.xAxis[0].setExtremes(null, null);
  chart.yAxis[0].setExtremes(null, null);
  showContextMenu.value = false;
}

const handleDeletePoints = () => {
  const chart = getCurrentChart();

  if (selectedIds.value.length === 0) {
    return;
  }

  selectedIds.value.forEach(id => {
    const point = chart.series[0]?.data.find(point => point.id === id);
    if (point) {
      point.remove(true, true);
    }
  });

  resetContextMenu();
}

const handleRecalculatePCA = async () => {
  console.log('Recalculating PCA...');
  seriesData.value = await setupChartSeries();
  const chart = getCurrentChart();
  chart.series[0].setData(seriesData.value[0].data);
  resetContextMenu();
}
</script>

<style lang="scss" scoped>
  .chartWrapper {
    position: relative;

    .contextMenu {
      position: absolute;
      width: 380px;
      background: #fff;
      border-radius: 5px;
      box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
      top: 5%;
      left: 50%;
      transform: translateX(-50%);

      ul {
        list-style: none;
        margin: 0;
        padding: 0;

        li {
          padding: 10px 15px;
          cursor: pointer;
          display: flex;
          align-items: center;
          white-space: nowrap;
          gap: 5px;
          font-size: 14px;
          font-family: Arial, sans-serif;

          &.disabled {
            opacity: .3;
            cursor: not-allowed;
          }

          &.border-block {
            border-block: 1px solid rgba(0, 0, 0, 0.1);
          }
        }
      }
    }
  }
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