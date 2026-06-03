<script setup>
import { onMounted, ref, watch } from 'vue'
import * as d3 from 'd3'

// On renomme la prop en 'activeNode' pour coller exactement à ce que donne HomeView
const props = defineProps({
  activeNode: {
    type: Object,
    default: null
  }
})

const loading = ref(false)
const svgRef = ref(null)
const targetNodeId = ref(null)

onMounted(() => {
  if (props.activeNode && props.activeNode.id) {
    targetNodeId.value = String(props.activeNode.id)
    renderEgoNetwork()
  }
})

// On observe 'activeNode' à la place de 'selectedNode'
watch(() => props.activeNode, (newVal) => {
  if (newVal && newVal.id) {
    targetNodeId.value = String(newVal.id)
    renderEgoNetwork()
  }
}, { deep: true })

async function renderEgoNetwork() {
  if (!targetNodeId.value) return
  loading.value = true
  
  try {
    const response = await fetch('/MC1_graph.json')
    const data = await response.json()

    const centerId = targetNodeId.value

    const nodeTypeMap = new Map()
    const nodeNameMap = new Map()
    data.nodes.forEach(n => {
      if (n && n.id !== undefined) {
        nodeTypeMap.set(String(n.id), n['Node Type'] || 'Inconnu')
        nodeNameMap.set(String(n.id), n.name || 'Sans nom')
      }
    })

    if (!nodeTypeMap.has(centerId)) {
      loading.value = false
      return
    }

    const rawLinks = data.links || data.edges || []
    const connectedLinks = rawLinks.filter(l => {
      const sId = String(l.source?.id !== undefined ? l.source.id : l.source)
      const tId = String(l.target?.id !== undefined ? l.target.id : l.target)
      
      return (sId === centerId || tId === centerId) && nodeTypeMap.has(sId) && nodeTypeMap.has(tId)
    })

    const neighborIds = new Set()
    neighborIds.add(centerId)
    connectedLinks.forEach(l => {
      neighborIds.add(String(l.source?.id !== undefined ? l.source.id : l.source))
      neighborIds.add(String(l.target?.id !== undefined ? l.target.id : l.target))
    })

    const egoNodes = Array.from(neighborIds).map(id => ({
      id,
      name: nodeNameMap.get(id),
      'Node Type': nodeTypeMap.get(id)
    }))

    const egoLinks = connectedLinks.map(l => ({
      source: String(l.source?.id !== undefined ? l.source.id : l.source),
      target: String(l.target?.id !== undefined ? l.target.id : l.target),
      type: l['Edge Type'] || l.type || 'Lien'
    }))

    const width = 450
    const height = 280
    const svg = d3.select(svgRef.value)
      .attr('width', '100%')
      .attr('height', height)
      .attr('viewBox', `0 0 ${width} ${height}`)

    svg.selectAll('*').remove()
    
    // Ajout d'un groupe principal 'g' pour supporter le Zoom/Pan comme le grand graphe
    const g = svg.append('g').attr('class', 'ego-content')

    svg.call(d3.zoom().scaleExtent([0.5, 3]).on('zoom', (event) => {
      g.attr('transform', event.transform)
    }))

    const getColor = (type) => {
      switch(type) {
        case 'Person': return '#3b82f6'
        case 'Song': return '#10b981'
        case 'RecordLabel': return '#ef4444'
        default: return '#94a3b8'
      }
    }

    const simulation = d3.forceSimulation(egoNodes)
      .force('link', d3.forceLink(egoLinks).id(d => d.id).distance(65))
      .force('charge', d3.forceManyBody().strength(-200))
      .force('center', d3.forceCenter(width / 2, height / 2))

    const link = g.selectAll('line')
      .data(egoLinks)
      .join('line')
      .attr('stroke', '#cbd5e1')
      .attr('stroke-width', 2)

    const node = g.selectAll('circle')
      .data(egoNodes)
      .join('circle')
      .attr('r', d => d.id === centerId ? 12 : 7)
      .attr('fill', d => getColor(d['Node Type']))
      .attr('stroke', d => d.id === centerId ? '#1e293b' : '#ffffff')
      .attr('stroke-width', d => d.id === centerId ? 2.5 : 1.5)
      .style('cursor', 'pointer')

    const label = g.selectAll('text')
      .data(egoNodes)
      .join('text')
      .attr('dy', d => d.id === centerId ? -16 : -11)
      .attr('text-anchor', 'middle')
      .attr('font-family', 'sans-serif')
      .attr('font-size', d => d.id === centerId ? '11px' : '9px')
      .attr('font-weight', d => d.id === centerId ? 'bold' : '600')
      .attr('fill', '#334155')
      .text(d => d.name && d.name.length > 14 ? d.name.substring(0, 12) + '...' : d.name)

    simulation.on('tick', () => {
      link.attr('x1', d => d.source.x).attr('y1', d => d.source.y)
          .attr('x2', d => d.target.x).attr('y2', d => d.target.y)
      node.attr('cx', d => d.x).attr('cy', d => d.y)
      label.attr('x', d => d.x).attr('y', d => d.y)
    })

    loading.value = false
  } catch (err) {
    console.error("❌ Erreur Ego Network :", err)
    loading.value = false
  }
}
</script>

<template>
  <div class="p-2 bg-white rounded text-center">
    <p v-if="loading" class="text-xs text-amber-600 animate-pulse py-16">
      Extraction du voisinage réseau...
    </p>
    <div v-else-if="!targetNodeId" class="text-xs text-slate-400 py-24 flex flex-col items-center justify-center font-medium">
      <span class="text-2xl mb-1">🕵️‍♂️</span>
      <span>En attente de cible</span>
      <p class="text-[10px] mt-1 max-w-[200px] text-slate-400 font-normal">
        Cliquez sur une entité du diagramme principal pour cartographier ses relations au 1er degré.
      </p>
    </div>
    <div v-else>
      <div class="text-[9px] font-bold text-slate-400 uppercase tracking-wider mb-2 text-left px-1">
        Voisinage direct de l'entité ciblée (1er degré)
      </div>
      <svg ref="svgRef" class="w-full bg-slate-50/50 rounded-xl border border-slate-100 min-h-[250px] cursor-move"></svg>
    </div>
  </div>
</template>