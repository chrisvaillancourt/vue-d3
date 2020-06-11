<template>
  <div class="example">
    <h1>Bar Chart</h1>
    <div>
      <BarChart></BarChart>
    </div>
  </div>
</template>

<script>
import BarChart from '../components/BarChart.vue';
export default {
  name: 'BarChartExample',
  components: {
    BarChart,
  },
};
</script>

<style scoped>
.example {
  /* position: relative; */
  /* display: flex; */
  /* flex-direction: column; */
  /* justify-content: center; */
}
</style>
