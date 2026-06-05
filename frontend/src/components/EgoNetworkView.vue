<template>
  <div class="ego-network-container w-full min-h-[160px] relative flex flex-col justify-center items-center">
    
    <div 
      v-if="!linksForEgo || linksForEgo.length === 0" 
      class="text-center p-4 bg-slate-50 rounded-lg border border-dashed border-slate-200 w-full"
    >
      <span class="text-2xl block mb-1">🔗</span>
      <p class="text-xs font-semibold text-slate-700">Aucune relation détectée</p>
      <p class="text-[11px] text-slate-400 mt-0.5">
        L'entité <span class="font-medium text-slate-600">"{{ selectedNode?.name || 'Inconnue' }}"</span> n'a pas de relations directes actives avec les filtres actuels.
      </p>
    </div>

    <svg 
      v-show="linksForEgo && linksForEgo.length > 0" 
      ref="egoSvg" 
      class="w-full bg-slate-50 rounded-lg" 
      height="160"
    ></svg>

  </div>
</template>

<script setup>
import { ref, computed, watch, onMounted, nextTick } from 'vue';
import * as d3 from 'd3';

const props = defineProps({
  selectedNode: {
    type: Object,
    default: null
  },
  allLinks: {
    type: Array,
    default: () => []
  }
});

const egoSvg = ref(null);

// Filtrage robuste : Extrait et compare les IDs qu'ils soient numériques ou objets D3
const linksForEgo = computed(() => {
  if (!props.selectedNode) return [];
  
  // Récupère l'ID du nœud sélectionné de manière propre
  const targetId = typeof props.selectedNode === 'object' && props.selectedNode.id !== undefined 
    ? Number(props.selectedNode.id) 
    : Number(props.selectedNode);
  
  return props.allLinks.filter(l => {
    // Extraction de l'ID source (s'adapte si D3 a transformé le lien en objet ou s'il est resté un ID brut)
    const sourceId = l.source && typeof l.source === 'object' ? Number(l.source.id) : Number(l.source);
    // Extraction de l'ID target
    const targetIdRef = l.target && typeof l.target === 'object' ? Number(l.target.id) : Number(l.target);
    
    return sourceId === targetId || targetIdRef === targetId;
  });
});

const drawEgoNetwork = async () => {
  await nextTick();
  
  // Si pas de SVG ou aucun lien trouvé pour ce nœud, on arrête et le v-if gère l'affichage du message
  if (!egoSvg.value || linksForEgo.value.length === 0) return;

  const svg = d3.select(egoSvg.value);
  svg.selectAll("*").remove(); // Nettoie impérativement le dessin précédent

  const width = egoSvg.value.clientWidth || 240;
  const height = 160;

  const nodesMap = new Map();
  const currentCenterId = typeof props.selectedNode === 'object' ? Number(props.selectedNode.id) : Number(props.selectedNode);
  
  // 1. Ajouter le nœud central sélectionné
  nodesMap.set(currentCenterId, { ...props.selectedNode, id: currentCenterId, isCenter: true });

  // 2. Parcourir les liens pour ajouter les nœuds voisins directs avec leurs vraies métadonnées si disponibles
  linksForEgo.value.forEach(l => {
    const sId = l.source && typeof l.source === 'object' ? Number(l.source.id) : Number(l.source);
    const tId = l.target && typeof l.target === 'object' ? Number(l.target.id) : Number(l.target);
    
    if (!nodesMap.has(sId)) {
      nodesMap.set(sId, { id: sId, name: l.source?.name || `Entité_${sId}`, type: l.source?.type || 'Person' });
    }
    if (!nodesMap.has(tId)) {
      nodesMap.set(tId, { id: tId, name: l.target?.name || `Entité_${tId}`, type: l.target?.type || 'Person' });
    }
  });

  const nodes = Array.from(nodesMap.values());
  
  // Normaliser les liens pour D3 (clonage pour éviter de polluer le store parent)
  const links = linksForEgo.value.map(l => {
    const sId = l.source && typeof l.source === 'object' ? Number(l.source.id) : Number(l.source);
    const tId = l.target && typeof l.target === 'object' ? Number(l.target.id) : Number(l.target);
    return { source: sId, target: tId };
  });

  // Simulation D3 locale
  const simulation = d3.forceSimulation(nodes)
    .force("link", d3.forceLink(links).id(d => d.id).distance(40))
    .force("charge", d3.forceManyBody().strength(-120))
    .force("center", d3.forceCenter(width / 2, height / 2));

  // Rendu des arêtes (liens)
  const linkElements = svg.append("g")
    .attr("stroke", "#cbd5e1")
    .attr("stroke-width", 1.5)
    .selectAll("line")
    .data(links)
    .join("line");

  // Rendu des nœuds (cercles)
  const nodeElements = svg.append("g")
    .selectAll("circle")
    .data(nodes)
    .join("circle")
    .attr("r", d => d.isCenter ? 8 : 5)
    .attr("fill", d => {
      if (d.isCenter) return '#2563eb'; // Bleu pour le nœud cliqué au centre
      if (d.type === 'Song') return '#10b981'; // Vert
      if (d.type === 'RecordLabel' || d.type === 'Label') return '#f43f5e'; // Rouge
      return '#64748b';
    })
    .attr("stroke", "#ffffff")
    .attr("stroke-width", 1);

  // Animation des forces
  simulation.on("tick", () => {
    linkElements
      .attr("x1", d => d.source.x)
      .attr("y1", d => d.source.y)
      .attr("x2", d => d.target.x)
      .attr("y2", d => d.target.y);

    nodeElements
      .attr("cx", d => d.x)
      .attr("cy", d => d.y);
  });
};

// CORRECTION ICI : On observe selectedNode ET linksForEgo pour capter les clics ET les changements de filtres
watch(
  [() => props.selectedNode, () => linksForEgo.value],
  () => {
    drawEgoNetwork();
  },
  { deep: true, immediate: true }
);

onMounted(() => {
  setTimeout(drawEgoNetwork, 200);
});
</script>