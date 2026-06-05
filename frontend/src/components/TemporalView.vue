<template>
  <div class="w-full">
    <svg ref="temporalSvg" class="w-full" height="105"></svg>
  </div>
</template>

<script setup>
import { ref, onMounted, nextTick } from 'vue';
import * as d3 from 'd3';

const props = defineProps({ selectedNode: Object });
const temporalSvg = ref(null);

onMounted(async () => {
  await nextTick();
  if (!temporalSvg.value) return;

  const svg = d3.select(temporalSvg.value);
  svg.selectAll("*").remove();

  const width = temporalSvg.value.clientWidth || 120;
  const height = 105;
  const margin = { top: 15, right: 10, bottom: 20, left: 20 };

  const peak = props.selectedNode?.peakYear || 2028;

  const timeline = [
    { year: 2023, value: 15 },
    { year: 2028, value: peak === 2028 ? 90 : 35 },
    { year: 2034, value: peak > 2030 ? 85 : 40 },
    { year: 2040, value: 20 }
  ];

  const x = d3.scaleLinear().domain([2023, 2040]).range([margin.left, width - margin.right]);
  const y = d3.scaleLinear().domain([0, 100]).range([height - margin.bottom, margin.top]);

  const line = d3.line()
    .x(d => x(d.year))
    .y(d => y(d.value))
    .curve(d3.curveMonotoneX);

  svg.append("path")
    .datum(timeline)
    .attr("fill", "none")
    .attr("stroke", "#10b981")
    .attr("stroke-width", 2)
    .attr("d", line);

  svg.append("g")
    .attr("transform", `translate(0,${height - margin.bottom})`)
    .call(d3.axisBottom(x).ticks(3).tickFormat(d3.format("d")))
    .style("font-size", "8px");
});
</script>