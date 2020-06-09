<template>
  <div>
    <h1>I'm a bar chart</h1>
    <div>
      <BarChart></BarChart>
    </div>
  </div>
</template>

<script>
import BarChart from "../components/BarChart.vue";
export default {
  name: "BarChartExample",
  components: {
    BarChart
  }
};
</script>

<style scoped></style>
