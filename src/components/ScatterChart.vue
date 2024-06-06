<template>
  <div class="chartWrapper">
    <div ref="chartContainer"></div>
    <div class="contextMenu" v-if="showContextMenu">
      <ul>
        <li :class="selectedIds.length ? 'active' : 'disabled'">
          <img src="/images/upload-big-arrow.png" width="12px" alt="">
          <span @click="handleContextMenu">Exclude</span>
        </li>
        <li :class="selectedIds.length ? 'active' : 'disabled'">
          <img src="/images/download.png" width="12px" alt="">
          <span  @click="handleContextMenu">Include</span>
        </li>
        <li @click="toggleSelectMode">
          <img src="/images/refresh-page-option.png" width="12px" alt="">
          <span>Actual action: "Select Mode"</span>
        </li>
        <li :class="selectedIds.length ? 'active' : 'disabled'">
          <img src="/images/change.png" width="12px" alt="">
          <span>Change selected spectra to class:</span>
          <select>
            <option value="0">Select an option</option>
            <option value="1">Class 1</option>
            <option value="2">Class 2</option>
            <option value="3">Class 3</option>
          </select>
        </li>
        <li>
          <img src="/images/magnifying-glass.png" width="12px" alt="">
          <span @click="handleResetZoom">Reset zoom</span>
        </li>
        <li :class="selectedIds.length ? 'active' : 'disabled'">
          <img src="/images/trash-can.png" width="12px" alt="">
          <span>Delete selected spectra</span>
        </li>
        <li class="border-block">
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
const showContextMenu = ref(false  );
const selectedIds = ref([]);
const seriesData = ref([]);

onMounted(async () => {
  try{
    const objectSpectroscopy = await axios.get('/data/objeto_espectros.json');
    seriesData.value = jsonScores.map(serie => ({
      name: serie.name,
      data: serie.data.map(point => ({
        id: point.fullidA,
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
          allowPointSelect: false,
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
    console.error('Error loading data:', error);
  }
});

// Sync hovered point with hoveredPointIndex
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

// Sync selected point with selectedPointIndex
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

// Show context menu
const handleContextMenu = (e) => {
  const chart = Highcharts.charts[props.chartId - 1]
  if (!selectedIds.value.length) {
    e.preventDefault();
    return;
  }

  if (e.target.innerText === 'Exclude' && selectedIds.value.length) {
    handleExclude(chart);
  }

  if (e.target.innerText === 'Include' && selectedIds.value.length) {
    handleInclude(chart);
  }

  selectedIds.value = [];
  showContextMenu.value = false;
}

const handleInclude = (chart) => {
  selectedIds.value.forEach(id => {
    const point = chart.series[0].data.find(point => point.id === id);
    point.update({
      color: "#00e272",
      selected: false
    });
  });
  chart.redraw();
}

const handleExclude = (chart) => {
  selectedIds.value.forEach(id => {
    const point = chart.series[0].data.find(point => point.id === id);
    point.update({
      color: 'rgba(0, 0, 0, 0.1)',
      selected: false
    });
  });
}

const handleCloseContextMenu = () => {
  showContextMenu.value = false;
}

const toggleSelectMode = () => {
  const chart = Highcharts.charts[props.chartId - 1]

  if (chart.series[0].options.allowPointSelect) {
    chart.series[0].update({
      allowPointSelect: false
    });
    selectedIds.value = [];

  } else {
    chart.series[0].update({
      allowPointSelect: true
    });
  }
  showContextMenu.value = false;
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