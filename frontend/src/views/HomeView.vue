<template>
  <div class="p-6 bg-slate-50 min-h-screen font-sans">
    <!-- En-tête de l'application -->
    <header class="mb-6">
      <h1 class="text-2xl font-bold text-slate-800">VAST KG - Dashboard Workspace</h1>
      <p class="text-xs text-slate-500">Outil d'analyse visuelle pour le journaliste Silas Reed (Période 2023 - 2040)</p>
    </header>

    <!-- Barre de Filtres Réactifs (Filter Feedback Loop) -->
    <div class="bg-white p-4 rounded-xl shadow-sm border border-slate-200 mb-6 flex flex-wrap gap-6 items-center justify-between">
      <div class="flex items-center gap-4">
        <span class="text-xs font-bold text-slate-700 uppercase tracking-wider">Filtres du Knowledge Graph :</span>
        <label class="inline-flex items-center gap-2 text-sm font-medium text-slate-700 cursor-pointer">
          <input type="checkbox" v-model="filters.personnes" class="rounded text-blue-600" />
          Personnes
        </label>
        <label class="inline-flex items-center gap-2 text-sm font-medium text-slate-700 cursor-pointer">
          <input type="checkbox" v-model="filters.chansons" class="rounded text-emerald-600" />
          Chansons
        </label>
        <label class="inline-flex items-center gap-2 text-sm font-medium text-slate-700 cursor-pointer">
          <input type="checkbox" v-model="filters.labels" class="rounded text-rose-600" />
          Labels
        </label>
      </div>
      <div class="text-xs font-semibold text-slate-500 bg-slate-100 px-3 py-1 rounded-full">
        Réseau Actif : {{ filteredNodes.length }} nœuds visibles
      </div>
    </div>

    <!-- Grille Principale : 2/3 pour les grands graphiques, 1/3 pour les analyses ciblées -->
    <div class="grid grid-cols-1 lg:grid-cols-3 gap-6">
      
      <!-- COLONNE GAUCHE & CENTRE : Visualisations globales -->
      <div class="lg:col-span-2 flex flex-col gap-6">
        
        <!-- Node-Link Diagram Global Unique (Sans panneau latéral intégré ni flottant) -->
        <div class="bg-white p-4 rounded-xl shadow-sm border border-slate-200">
          <h3 class="text-xs font-bold text-slate-400 uppercase tracking-wider mb-3">Graph Visualization // Node-Link Diagram</h3>
          <div class="border border-slate-100 rounded-lg p-2 bg-slate-50">
            <GraphView :nodes="filteredNodes" :links="filteredLinks" @node-click="handleNodeClick" />
          </div>
        </div>

        <!-- Sankey Diagram -->
        <div class="bg-white p-4 rounded-xl shadow-sm border border-slate-200">
          <h3 class="text-xs font-bold text-slate-400 uppercase tracking-wider mb-3">Graph Visualization // Sankey Diagram</h3>
          <SankeyView :links="filteredLinks" :key="filteredLinks.length" />
        </div>
      </div>

      <!-- COLONNE DE DROITE : Libérée, aérée et ordonnée -->
      <div class="flex flex-col gap-6">
        
        <!-- Neighborhood Exploration / Ego Network -->
        <div class="bg-white p-4 rounded-xl shadow-sm border border-slate-200">
          <h3 class="text-xs font-bold text-slate-400 uppercase tracking-wider mb-2">Neighborhood Exploration // Ego Network</h3>
          <EgoNetworkView :selectedNode="selectedNode" :allLinks="mockLinks" />
        </div>

        <!-- Community Discovery / Connected Components -->
        <div class="bg-white p-4 rounded-xl shadow-sm border border-slate-200">
          <h3 class="text-xs font-bold text-slate-400 uppercase tracking-wider mb-3">Community Discovery // Connected Components</h3>
          <ConnectedComponents :nodes="filteredNodes" :key="filteredNodes.length" />
        </div>

        <!-- Projections Spatiales et Temporelles -->
        <div class="grid grid-cols-2 gap-4">
          <div class="bg-white p-3 rounded-xl shadow-sm border border-slate-200">
            <h3 class="text-[10px] font-bold text-slate-400 uppercase tracking-wider mb-2">Spatial View (?)</h3>
            <SpatialView :selectedNode="selectedNode" :key="selectedNode ? selectedNode.id : 'none'" />
          </div>
          <div class="bg-white p-3 rounded-xl shadow-sm border border-slate-200">
            <h3 class="text-[10px] font-bold text-slate-400 uppercase tracking-wider mb-2">Temporal View (?)</h3>
            <TemporalView :selectedNode="selectedNode" :key="selectedNode ? selectedNode.id : 'none'" />
          </div>
        </div>

      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue';
import GraphView from '../components/GraphView.vue';
import SankeyView from '../components/SankeyView.vue';
import EgoNetworkView from '../components/EgoNetworkView.vue';
import ConnectedComponents from '../components/ConnectedComponents.vue';
import SpatialView from '../components/SpatialView.vue';
import TemporalView from '../components/TemporalView.vue';

// Configuration des filtres réactifs
const filters = ref({ personnes: true, chansons: true, labels: true });

// Sélection initiale par défaut
const selectedNode = ref({ id: 39, name: 'Healthier Than Yesterday', type: 'Song', zone: 'Coastal', peakYear: 2028 });

// Dataset mock VAST MC1 robuste
const mockNodes = Array.from({ length: 245 }, (_, i) => {
  let type = 'Person';
  if (i % 3 === 1) type = 'Song';
  if (i % 3 === 2) type = 'RecordLabel';
  
  const zones = ['Coastal', 'Capital City', 'Ports'];
  return {
    id: i + 1,
    name: i === 38 ? 'Healthier Than Yesterday' : `Entité_${i + 1}`,
    type: type,
    zone: zones[i % 3],
    peakYear: 2023 + (i % 18)
  };
});

const mockLinks = Array.from({ length: 240 }, (_, i) => ({
  source: (i % 245) + 1,
  target: ((i * 2 + 11) % 245) + 1,
  type: 'LINKED_TO'
}));

const filteredNodes = computed(() => {
  return mockNodes.filter(n => {
    if (n.type === 'Person' && !filters.value.personnes) return false;
    if (n.type === 'Song' && !filters.value.chansons) return false;
    if (n.type === 'RecordLabel' && !filters.value.labels) return false;
    return true;
  });
});

const filteredLinks = computed(() => {
  const activeIds = new Set(filteredNodes.value.map(n => n.id));
  return mockLinks.filter(l => activeIds.has(l.source) && activeIds.has(l.target));
});

const handleNodeClick = (node) => {
  selectedNode.value = node;
};
</script>