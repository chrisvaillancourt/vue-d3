<template>
  <div id="wrapper" class="wrapper">
    <div id="tooltip" class="tooltip">
      <div class="tooltip-range">Humidity: <span id="range"></span></div>
      <div class="tooltip-value"><span id="count"></span> days</div>
    </div>
  </div>
</template>

<script>
import * as d3 from "d3";
export default {
  name: "BarChart",
  data() {
    return {};
  },
  beforeMount() {
    this.drawBars().catch(console.error);
  },
  computed: {
    // weatherUrl() {
    //   return require("./static/data/nyc_weather_data.json");
    // }
  },
  methods: {
    async drawBars() {
      console.clear();
      console.log("drawing");
      // 1. Access data

      const dataset = await d3.json("./static/data/nyc_weather_data.json");

      // 2. Create chart dimensions
      function createDimensions({ customDimensions = {} } = {}) {
        return {
          margin: {
            top: 10,
            right: 10,
            bottom: 50,
            left: 10
          },
          get width() {
            return (
              (window.innerWidth - this.margin.left - this.margin.right) * 0.95
            );
          },
          get boundedWidth() {
            return this.width - this.margin.left - this.margin.right;
          },
          get height() {
            return this.width;
          },
          get boundedHeight() {
            return this.height - this.margin.top - this.margin.bottom;
          },
          ...customDimensions
        };
      }

      var dimensions = createDimensions();

      // 3. Draw canvas

      const wrapper = d3
        .select("#wrapper")
        .append("svg")
        .attr("width", dimensions.width)
        .attr("height", dimensions.height);

      const bounds = wrapper
        .append("g")
        .style(
          "transform",
          `translate(${dimensions.margin.left}px, ${dimensions.margin.top}px)`
        );

      // init static elements
      bounds.append("g").attr("class", "bins");
      bounds.append("line").attr("class", "mean");
      bounds
        .append("g")
        .attr("class", "x-axis")
        .style("transform", `translateY(${dimensions.boundedHeight}px)`)
        .append("text")
        .attr("class", "x-axis-label");

      const metricAccessor = d => d.humidity;
      const yAccessor = d => d.length;

      // 4. Create scales

      const xScale = d3
        .scaleLinear()
        .domain(d3.extent(dataset, metricAccessor))
        .range([0, dimensions.boundedWidth])
        .nice();

      const binsGenerator = d3
        .histogram()
        .domain(xScale.domain())
        .value(metricAccessor)
        .thresholds(12);

      const bins = binsGenerator(dataset);

      const yScale = d3
        .scaleLinear()
        .domain([0, d3.max(bins, yAccessor)])
        .range([dimensions.boundedHeight, 0])
        .nice();

      // 5. Draw data

      const barPadding = 1;

      let binGroups = bounds
        .select(".bins")
        .selectAll(".bin")
        .data(bins);

      binGroups.exit().remove();

      const newBinGroups = binGroups
        .enter()
        .append("g")
        .attr("class", "bin");

      newBinGroups.append("rect");
      newBinGroups.append("text");

      // update binGroups to include new points
      binGroups = newBinGroups.merge(binGroups);

      const barRects = binGroups
        .select("rect")
        .attr("x", d => xScale(d.x0) + barPadding)
        .attr("y", d => yScale(yAccessor(d)))
        .classed("rects", true)
        .attr("height", d => dimensions.boundedHeight - yScale(yAccessor(d)))
        .attr("width", d =>
          d3.max([0, xScale(d.x1) - xScale(d.x0) - barPadding])
        );

      const mean = d3.mean(dataset, metricAccessor);

      const meanLine = bounds
        .selectAll(".mean")
        .attr("x1", xScale(mean))
        .attr("x2", xScale(mean))
        .attr("y1", -20)
        .attr("y2", dimensions.boundedHeight);

      // draw axes
      const xAxisGenerator = d3.axisBottom().scale(xScale);

      const xAxis = bounds.select(".x-axis").call(xAxisGenerator);

      const xAxisLabel = xAxis
        .select(".x-axis-label")
        .attr("x", dimensions.boundedWidth / 2)
        .attr("y", dimensions.margin.bottom - 10)
        .text("Humidity");

      // 7. Set up interactions
      var tooltip = d3.select("#tooltip");

      // create text formatting function
      const formatHumidity = d3.format(".2f");
      // standard convention is to use id's in JS and
      // classes in CSS
      binGroups
        .select("rect")
        .on("mouseenter", handleMouseEnter)
        .on("mouseleave", handleMouseLeave);

      function handleMouseEnter(datum) {
        tooltip.select("#count").text(yAccessor(datum));
        tooltip
          .select("#range")
          .text([formatHumidity(datum.x0), formatHumidity(datum.x1)].join("-"));

        // to calc our tooltip's x position, we need to take 3 things into account:
        // 1) the bar's x position in the chart (xScale(datum.x0))
        // 2) half of the bar's width ((xScale(datum.x1)-xScale(datum.x0))/2)
        // 3) the margin by which the bounds are shifted right (dimensions.margin.left)

        const x =
          xScale(datum.x0) +
          (xScale(datum.x1) - xScale(datum.x0)) / 2 +
          dimensions.margin.left;
        // to calc our tooltip's y position, we need add:
        // 1) the bar's y position (yScale(yAccessor(datum)))
        // 2) the margin by which our bounds are shiften down (dimensions.margin.top)

        const y = yScale(yAccessor(datum)) + dimensions.margin.top;
        // we're setting a transform CSS property rather than a 'left' and 'top'
        // value because of the performance cost.
        // A good guide is to only directly change transforma and opacity

        // We also need to take the size of the tooltip into account without the cost of
        // calling .getBoundingClientRect().
        // transform: translate() can take a percentage value for the currently specified element
        tooltip
          .style(
            "transform",
            `translate(calc(-50% + ${x}px), calc(-100% + ${y}px))`
          )
          .style("opacity", 1);
      }
      function handleMouseLeave() {
        tooltip.style("opacity", 0);
      }
    }
  }
};
</script>

<style scoped>
.wrapper {
  /* set position on the wrapper to accommodate tooltip position */
  position: relative;
  /* display: grid; */
  display: flex;
  justify-content: center;
  /* margin: 5em 2em; */
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
  opacity: 0;
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
  content: "";
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