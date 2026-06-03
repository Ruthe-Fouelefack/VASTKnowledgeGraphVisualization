<script setup>
import { ref } from 'vue'
import DashboardCard from '../components/DashboardCard.vue'
import GraphView from '../components/GraphView.vue'
import SankeyView from '../components/SankeyView.vue'
import EgoView from '../components/EgoView.vue' 
import EgoNetworkView from '../components/EgoNetworkView.vue'

// Stocke l'élément cliqué dans le graphe principal
const selectedNodeForEgo = ref(null)

const handleNodeSelected = (node) => {
  selectedNodeForEgo.value = node
}

const cards = [
  {
    title: 'Connected components',
    subtitle: 'View for community discovery/connected components',
    content: 'High-level overview of the connected components in the knowledge graph.',
  },
  {
    title: 'Spatial/Geographic view (?)',
    subtitle: 'Geographical projection of data',
    content: 'Placeholder for a spatial or geographic view.',
  },
  {
    title: 'Temporal view (?)',
    subtitle: 'Temporal projection of data',
    content: 'Placeholder for a temporal view.',
  }
]
</script>

<template>
  <section class="grid grid-cols-1 gap-4 sm:grid-cols-2 xl:grid-cols-3">
    
    <div class="sm:col-span-2 xl:col-span-3">
      <DashboardCard title="Node-link diagram" subtitle="Graph Visualization">
        <GraphView @node-click="handleNodeSelected" />
      </DashboardCard>
    </div>

    <DashboardCard title="Sankey Diagram" subtitle="Graph Visualization">
      <SankeyView />
    </DashboardCard>

    <DashboardCard title="Ego Network" subtitle="Neighborhood exploration from a node">
      <EgoView :activeNode="selectedNodeForEgo" />
    </DashboardCard>

    <DashboardCard v-for="card in cards" :key="card.title" :title="card.title" :subtitle="card.subtitle">
      {{ card.content }}
    </DashboardCard>

  </section>
</template>