<template>
  <div id="wrapper" class="wrapper">
    <div
      :style="tooltip.styleObj"
      v-show="tooltip.display"
      id="tooltip"
      class="tooltip"
    >
      <div class="tooltip-range">
        Humidity: <span id="range">{{ tooltip.range }}</span>
      </div>
      <div class="tooltip-value">
        <span id="count">{{ tooltip.frequency }}</span> days
      </div>
    </div>
    <svg
      class="histogram"
      :height="dimensions.height"
      :width="dimensions.width"
    >
      <g
        class="bounds"
        :transform="
          `translate(${dimensions.margin.left}, ${dimensions.margin.top})`
        "
      >
        <g class="bins">
          <g v-for="d in bars" :key="d.id" class="bin">
            <rect
              v-on="{
                mouseenter: handleMouseEnter,
                mouseleave: handleMouseLeave,
              }"
              :x="d.x"
              :y="d.y"
              :height="d.height"
              :width="d.width"
              :data-tooltip-range="d.tooltipRange"
              :data-tooltip-frequency="d.frequency"
            ></rect>
            <text :x="d.label.x" :y="d.label.y">{{ d.label.text }}</text>
          </g>
        </g>
        <line
          :x1="line.x"
          :x2="line.x"
          :y1="line.y1"
          :y2="line.y2"
          stroke="maroon"
          stroke-dasharray="2px 4px"
        ></line>
        <text class="mean-text" :x="line.x" y="">mean</text>
        <g
          ref="xAxis"
          class="x-axis"
          :transform="`translate(0,${dimensions.boundedHeight})`"
        >
          <text
            class="x-axis-label"
            :x="dimensions.boundedWidth / 2"
            :y="dimensions.margin.bottom - 10"
          >
            Humidity
          </text>
        </g>
      </g>
    </svg>
  </div>
</template>

<script>
import { json } from 'd3-fetch';
import { select } from 'd3-selection';
import { scaleLinear } from 'd3-scale';
import { extent, max, histogram, mean } from 'd3-array';
import { axisBottom } from 'd3-axis';
import { format } from 'd3-format';
export default {
  name: 'BarChart',
  data() {
    return {
      dimensions: {
        margin: {
          top: 10,
          right: 10,
          bottom: 50,
          left: 10,
        },
        width: null,
        boundedWidth: null,
        height: null,
        boundedHeight: null,
      },
      weatherData: [],
      xScale: function() {},
      yScale: function() {},
      formatMetric: format('.2f'),
      bars: [],
      tooltip: {
        display: false,
        range: '',
        frequency: '',
        styleObj: {
          transform: ``,
        },
      },
    };
  },
  beforeMount() {
    this.loadData().catch(console.error);
  },
  mounted() {
    this.updateDimensions({
      newHeight: window.innerHeight * 0.5,
      newWidth: window.innerWidth * 0.5,
    });
    window.addEventListener('resize', this.handleResize);
  },
  destroyed() {
    window.removeEventListener('resize', this.handleResize);
    console.clear();
  },
  computed: {
    dataMean: function() {
      return mean(this.weatherData, this.metricAccessor);
    },
    line: function() {
      return {
        x: this.xScale(this.dataMean) || 0,
        y1: 10,
        y2: this.dimensions.boundedHeight,
      };
    },
  },
  watch: {
    weatherData: function() {
      this.calculateScales();
      this.renderAxis();
    },
  },
  methods: {
    updateDimensions({ newHeight, newWidth } = {}) {
      this.dimensions.width = newWidth;
      this.dimensions.height = newHeight;
      this.dimensions.boundedWidth =
        newWidth - this.dimensions.margin.left - this.dimensions.margin.right;
      this.dimensions.boundedHeight =
        newHeight - this.dimensions.margin.top - this.dimensions.margin.bottom;
    },
    handleResize(e) {
      this.updateDimensions({
        newHeight: window.innerHeight * 0.5,
        newWidth: window.innerWidth * 0.5,
      });
    },
    async loadData() {
      this.weatherData = await json('./static/data/nyc_weather_data.json');
    },
    metricAccessor(d) {
      return d.humidity;
    },
    yAccessor(d) {
      // length of the bins == number of values in that group (frequency)
      return d.length;
    },
    calculateScales() {
      if (!this.weatherData.length) return;

      this.xScale = scaleLinear()
        .domain(extent(this.weatherData, this.metricAccessor))
        .range([0, this.dimensions.boundedWidth])
        .nice();
      var binsGenerator = histogram()
        .domain(this.xScale.domain())
        .value(this.metricAccessor)
        .thresholds(12);

      var bins = binsGenerator(this.weatherData).filter(d => {
        // only include bins with a frequency > 0
        return d.length > 0;
      });

      this.yScale = scaleLinear()
        .domain([0, max(bins, this.yAccessor)])
        .range([this.dimensions.boundedHeight, 0])
        .nice();
      var barPadding = 1;
      // create bars
      this.bars = bins.map((d, i) => {
        var { x0, x1 } = d;
        var x = this.xScale(x0) + barPadding;
        var labelX = this.xScale(x0) + (this.xScale(x1) - this.xScale(x0)) / 2;
        var labelY = this.yScale(this.yAccessor(d)) - 5;
        var y = this.yScale(this.yAccessor(d));
        var height =
          this.dimensions.boundedHeight - this.yScale(this.yAccessor(d));
        var width = max([0, this.xScale(x1) - this.xScale(x0) - barPadding]);
        var frequency = this.yAccessor(d);
        var tooltipRange = [this.formatMetric(x0), this.formatMetric(x1)].join(
          '-'
        );
        return {
          x,
          y,
          height,
          width,
          id: i,
          tooltipRange,
          frequency,
          label: {
            x: labelX,
            y: labelY,
            text: frequency,
          },
        };
      });
    },
    renderAxis() {
      var xAxisGenerator = axisBottom().scale(this.xScale);
      select(this.$refs.xAxis).call(xAxisGenerator);
    },
    handleMouseEnter(e) {
      var { target } = e;

      this.tooltip.frequency = target.dataset.tooltipFrequency;
      this.tooltip.range = target.dataset.tooltipRange;

      var x = Number(target.getAttribute('x'));

      // var x2 =
      //   this.xScale(datum.x0) +
      //   (xScale(datum.x1) - xScale(datum.x0)) / 2 +
      //   dimensions.margin.left;

      var y = Number(target.getAttribute('y'));
      var width = Number(target.getAttribute('width'));
      // console.log(width);
      this.tooltip.styleObj.transform = `translate(calc(${x +
        width / 2 +
        this.dimensions.margin.left}px), calc(-100% + ${y}px))`;

      this.tooltip.display = true;
    },
    handleMouseLeave() {
      this.tooltip.display = false;
    },
  },
};
</script>

<style scoped>
.mean-text {
  fill: maroon;
  text-anchor: middle;
}
.wrapper {
  /* set position on the wrapper to accommodate tooltip position */
  position: relative;
  display: flex;
  justify-content: center;
  font-family: sans-serif;
}
.bin rect {
  fill: cornflowerblue;
}
.bin rect:hover {
  fill: purple;
}

.bin text {
  text-anchor: middle;
  fill: darkgrey;
  font-size: 12px;
  font-family: sans-serif;
}

.mean {
  stroke: maroon;
  stroke-dasharray: 2px 4px;
}

.x-axis-label {
  fill: black;
  font-size: 1.4em;
  text-transform: capitalize;
}

.tooltip {
  /* opacity: 0; */
  position: absolute;
  top: -12px;
  left: 0;
  padding: 0.6em 1em;
  background: #fff;
  text-align: center;
  border: 1px solid #ddd;
  z-index: 10;
  transition: all 0.2s ease-out;
  pointer-events: none;
}

.tooltip:before {
  content: '';
  position: absolute;
  bottom: 0;
  left: 50%;
  width: 12px;
  height: 12px;
  background: white;
  border: 1px solid #ddd;
  border-top-color: transparent;
  border-left-color: transparent;
  transform: translate(-50%, 50%) rotate(45deg);
  transform-origin: center center;
  z-index: 10;
}

.tooltip-range {
  margin-bottom: 0.2em;
  font-weight: 600;
}
</style>
