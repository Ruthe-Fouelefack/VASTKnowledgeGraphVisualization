<template>
  <div class="w-full h-[120px]">
    <svg ref="ccSvg" class="w-full h-full"></svg>
  </div>
</template>

<script setup>
import { ref, onMounted, watch, nextTick } from 'vue';
import * as d3 from 'd3';

const props = defineProps({ nodes: Array });
const ccSvg = ref(null);

const drawCC = async () => {
  await nextTick();
  if (!ccSvg.value) return;

  const svg = d3.select(ccSvg.value);
  svg.selectAll("*").remove();

  const width = ccSvg.value.clientWidth || 250;
  const height = 120;
  const margin = { top: 15, right: 15, bottom: 25, left: 30 };

  const total = props.nodes?.length || 100;
  const dataset = [
    { name: 'Ivy Core', value: Math.round(total * 0.45) },
    { name: 'Indie Pop', value: Math.round(total * 0.30) },
    { name: 'Folk Gen', value: Math.round(total * 0.15) },
    { name: 'Outliers', value: Math.round(total * 0.10) }
  ];

  const x = d3.scaleBand().domain(dataset.map(d => d.name)).range([margin.left, width - margin.right]).padding(0.35);
  const y = d3.scaleLinear().domain([0, d3.max(dataset, d => d.value) || 10]).range([height - margin.bottom, margin.top]);

  svg.append("g")
    .attr("fill", "#4f46e5")
    .selectAll("rect")
    .data(dataset)
    .join("rect")
    .attr("x", d => x(d.name))
    .attr("y", d => y(d.value))
    .attr("height", d => y(0) - y(d.value))
    .attr("width", x.bandwidth())
    .attr("rx", 3);

  svg.append("g")
    .attr("transform", `translate(0,${height - margin.bottom})`)
    .call(d3.axisBottom(x))
    .style("font-size", "8px");

  svg.append("g")
    .attr("transform", `translate(${margin.left},0)`)
    .call(d3.axisLeft(y).ticks(3))
    .style("font-size", "8px");
};

watch(() => props.nodes, () => drawCC(), { deep: true });
onMounted(() => setTimeout(drawCC, 200));
</script>