<template>
  <div class="w-full text-center">
    <p class="text-[9px] text-slate-500 mb-1">Analyse d'ancrage (Secteurs d'Oceanus) :</p>
    <svg ref="spatialSvg" class="w-full" height="105"></svg>
  </div>
</template>

<script setup>
import { ref, onMounted, nextTick } from 'vue';
import * as d3 from 'd3';

const props = defineProps({ selectedNode: Object });
const spatialSvg = ref(null);

onMounted(async () => {
  await nextTick();
  if (!spatialSvg.value) return;

  const svg = d3.select(spatialSvg.value);
  svg.selectAll("*").remove();

  const width = spatialSvg.value.clientWidth || 120;
  const currentZone = props.selectedNode?.zone || 'Coastal';

  const regions = [
    { id: 'Coastal', cx: width * 0.22, cy: 55, r: 16, label: 'Coastal' },
    { id: 'Capital City', cx: width * 0.50, cy: 45, r: 22, label: 'Capital' },
    { id: 'Ports', cx: width * 0.78, cy: 60, r: 14, label: 'Ports' }
  ];

  const groups = svg.selectAll("g").data(regions).join("g");

  groups.append("circle")
    .attr("cx", d => d.cx)
    .attr("cy", d => d.cy)
    .attr("r", d => d.r)
    .attr("fill", d => d.id === currentZone ? "#f43f5e" : "#e2e8f0")
    .attr("stroke", d => d.id === currentZone ? "#e11d48" : "#cbd5e1")
    .attr("stroke-width", d => d.id === currentZone ? 2 : 1);

  groups.append("text")
    .attr("x", d => d.cx)
    .attr("y", d => d.cy + 3)
    .attr("text-anchor", "middle")
    .style("font-size", "8px")
    .style("fill", d => d.id === currentZone ? "#fff" : "#475569")
    .text(d => d.label);
});
</script>