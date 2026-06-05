<template>
  <div class="w-full p-2">
    <svg ref="statsSvg" class="w-full" height="110"></svg>
  </div>
</template>

<script setup>
import { ref, onMounted, watch, nextTick } from 'vue';
import * as d3 from 'd3';

const props = defineProps({
  nodes: {
    type: Array,
    default: () => []
  }
});

const statsSvg = ref(null);

const drawStats = async () => {
  await nextTick();
  if (!statsSvg.value) return;

  const svg = d3.select(statsSvg.value);
  
  // NETTOYAGE CRUCIAL : Supprime tout élément existant pour éviter le gel des barres graphiques
  svg.selectAll("*").remove();

  // Dimensions et marges de calcul
  const width = statsSvg.value.clientWidth || 260;
  const barHeight = 14;
  const barSpacing = 28;
  const labelOffset = 110; 
  const maxBarWidth = width - labelOffset - 35;

  // Extraction et filtrage robuste et insensible à la casse / pluriels
  const pCount = props.nodes.filter(n => {
    const t = String(n.type).toLowerCase();
    return t.includes('person') || t.includes('user');
  }).length;

  const sCount = props.nodes.filter(n => {
    const t = String(n.type).toLowerCase();
    return t.includes('song') || t.includes('chanson');
  }).length;

  const lCount = props.nodes.filter(n => {
    const t = String(n.type).toLowerCase();
    return t.includes('label') || t.includes('record');
  }).length;

  // Création du modèle de données local
  const counts = [
    { label: 'Personnes', count: pCount, color: '#3b82f6' },
    { label: 'Chansons', count: sCount, color: '#10b981' },
    { label: 'Labels de Disques', count: lCount, color: '#f43f5e' }
  ];

  // Définition de l'échelle linéaire dynamique
  const maxCount = d3.max(counts, d => d.count) || 1;
  const widthScale = d3.scaleLinear()
    .domain([0, maxCount])
    .range([0, maxBarWidth]);

  const chart = svg.append("g").attr("transform", "translate(10, 15)");

  // Construction des lignes du graphique
  const rows = chart.selectAll("g.row")
    .data(counts)
    .join("g")
    .attr("class", "row")
    .attr("transform", (d, i) => `translate(0, ${i * barSpacing})`);

  // Rendu du texte descriptif (Label de gauche)
  rows.append("text")
    .attr("x", 0)
    .attr("y", barHeight - 2)
    .style("font-size", "11px")
    .style("fill", "#475569")
    .style("font-weight", "500")
    .text(d => d.label);

  // Rendu de la barre horizontale réactive
  rows.append("rect")
    .attr("x", labelOffset)
    .attr("y", 0)
    .attr("width", d => widthScale(d.count))
    .attr("height", barHeight)
    .attr("fill", d => d.color)
    .attr("rx", 3);

  // Rendu de la valeur numérique courante (Compteur de droite)
  rows.append("text")
    .attr("x", d => labelOffset + widthScale(d.count) + 8)
    .attr("y", barHeight - 2)
    .style("font-size", "11px")
    .style("font-weight", "bold")
    .style("fill", "#1e293b")
    .text(d => d.count);
};

// Écouteur réactif profond sur la propriété "nodes" pour recalculer au moindre changement
watch(() => props.nodes, () => {
  drawStats();
}, { deep: true, immediate: true });

onMounted(() => {
  setTimeout(drawStats, 200);
});
</script>