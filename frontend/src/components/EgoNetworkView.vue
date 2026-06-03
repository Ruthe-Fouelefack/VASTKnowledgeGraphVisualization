<script setup>
import { onMounted, watch, ref } from 'vue'
import * as d3 from 'd3'

const props = defineProps({
  activeNode: {
    type: Object,
    default: null
  }
})

const loading = ref(false)
const svgRef = ref(null)
const allData = ref({ nodes: [], links: [] })
const noLinksFound = ref(false)
const internalError = ref(null)

// 1. Chargement initial du fichier JSON original
onMounted(async () => {
  try {
    loading.value = true
    const response = await fetch('/MC1_graph.json')
    const data = await response.json()
    
    // On clone les structures brutes sans aucune modification
    allData.value.nodes = (data.nodes || []).map(n => ({ ...n, id: String(n.id) }))
    allData.value.links = (data.links || data.edges || []).map(l => ({ ...l }))
    
    loading.value = false
    if (props.activeNode) drawEgoNetwork()
  } catch (error) {
    console.error("❌ Erreur d'initialisation JSON Ego:", error)
    loading.value = false
  }
})

// 2. On redessine dès que la prop activeNode change
watch(() => props.activeNode, () => {
  drawEgoNetwork()
})

// 3. Moteur de calcul du sous-graphe en étoile
const drawEgoNetwork = () => {
  if (!svgRef.value || !props.activeNode || !allData.value.nodes.length) return

  // Réinitialisation des états
  noLinksFound.value = false
  internalError.value = null
  
  const targetId = String(props.activeNode.id)
  
  // Dictionnaire pour retrouver instantanément un nœud par son ID
  const nodeMap = new Map()
  allData.value.nodes.forEach(n => nodeMap.set(n.id, n))

  const egoLinks = []
  const neighborIds = new Set()
  neighborIds.add(targetId)

  try {
    // Parcours ultra-sécurisé de chaque lien pour parer à toutes les structures D3 possibles
    allData.value.links.forEach(l => {
      if (!l) return

      let sId = ""
      let tId = ""

      // Extraction de la Source
      if (l.source !== null && typeof l.source === 'object') {
        sId = String(l.source.id !== undefined ? l.source.id : (l.source.index !== undefined ? l.source.index : ""))
      } else if (l.source !== undefined) {
        sId = String(l.source)
      }

      // Extraction de la Cible (Target)
      if (l.target !== null && typeof l.target === 'object') {
        tId = String(l.target.id !== undefined ? l.target.id : (l.target.index !== undefined ? l.target.index : ""))
      } else if (l.target !== undefined) {
        tId = String(l.target)
      }

      // Si le lien touche notre nœud cible, on vérifie la validité
      if (sId === targetId || tId === targetId) {
        // Est-ce que les nœuds existent bien dans le tableau de référence ?
        if (nodeMap.has(sId) && nodeMap.has(tId)) {
          neighborIds.add(sId)
          neighborIds.add(tId)
          egoLinks.push({
            source: sId,
            target: tId,
            type: l['Edge Type'] || l.type || 'Relation'
          })
        }
      }
    })

    // Si après examen aucun lien n'est retenu
    if (egoLinks.length === 0) {
      noLinksFound.value = true
      const svg = d3.select(svgRef.value)
      svg.selectAll('*').remove()
      return
    }

    // Récupération des objets nœuds complets
    const egoNodes = allData.value.nodes.filter(n => neighborIds.has(n.id)).map(n => ({ ...n }))

    // Dimensions du plan de rendu
    const width = 500
    const height = 280

    const svg = d3.select(svgRef.value)
      .attr('width', '100%')
      .attr('height', height)
      .attr('viewBox', `0 0 ${width} ${height}`)

    svg.selectAll('*').remove()

    // Lancement du simulateur de forces
    const simulation = d3.forceSimulation(egoNodes)
      .force('link', d3.forceLink(egoLinks).id(d => d.id).distance(75))
      .force('charge', d3.forceManyBody().strength(-140))
      .force('center', d3.forceCenter(width / 2, height / 2))

    const getColor = (type) => {
      switch(type) {
        case 'Person': return '#3b82f6'
        case 'Song': return '#10b981'
        case 'RecordLabel': return '#ef4444'
        default: return '#94a3b8'
      }
    }

    // Tracé des lignes
    const linkSelection = svg.append('g')
      .attr('stroke', '#cbd5e1')
      .attr('stroke-opacity', 0.8)
      .attr('stroke-width', 2)
      .selectAll('line')
      .data(egoLinks)
      .join('line')

    linkSelection.append('title').text(d => d.type)

    // Tracé des bulles
    const nodeSelection = svg.append('g')
      .selectAll('circle')
      .data(egoNodes)
      .join('circle')
      .attr('r', d => d.id === targetId ? 13 : 7)
      .attr('fill', d => getColor(d['Node Type']))
      .attr('stroke', d => d.id === targetId ? '#0f172a' : '#ffffff')
      .attr('stroke-width', d => d.id === targetId ? 3 : 1)

    nodeSelection.append('title').text(d => `${d.name}\nType: ${d['Node Type']}`)

    // Tracé des textes
    const labelSelection = svg.append('g')
      .attr('fill', '#334155')
      .attr('font-family', 'sans-serif')
      .attr('font-size', '9px')
      .attr('font-weight', '700')
      .selectAll('text')
      .data(egoNodes)
      .join('text')
      .attr('dy', d => d.id === targetId ? -17 : -12)
      .attr('text-anchor', 'middle')
      .text(d => d.name && d.name.length > 15 ? d.name.substring(0, 13) + '...' : d.name)

    // Actualisation cinétique des coordonnées
    simulation.on('tick', () => {
      linkSelection
        .attr('x1', d => d.source.x)
        .attr('y1', d => d.source.y)
        .attr('x2', d => d.target.x)
        .attr('y2', d => d.target.y)

      nodeSelection
        .attr('cx', d => d.x)
        .attr('cy', d => d.y)

      labelSelection
        .attr('x', d => d.x)
        .attr('y', d => d.y)
    })

  } catch (err) {
    console.error("❌ Erreur interne lors du dessin de l'Ego Network:", err)
    internalError.value = err.message
  }
}
</script>

<template>
  <div class="p-2 bg-white rounded text-center">
    <p v-if="loading" class="text-xs text-amber-600 animate-pulse py-16">
      Extraction des proximités relationnelles...
    </p>
    
    <div v-else>
      <div v-if="internalError" class="text-center py-16 text-red-500 font-mono text-[10px] bg-red-50 rounded-xl p-4">
        <p class="font-bold">⚠️ Erreur de structure détectée sur ce nœud :</p>
        <p class="mt-1 text-slate-600">{{ internalError }}</p>
      </div>

      <div v-else-if="noLinksFound" class="text-center py-16 text-slate-400 flex flex-col items-center justify-center min-h-[250px]">
        <span class="text-2xl mb-2">📭</span>
        <p class="text-xs font-semibold text-slate-600">🎯 Cible : {{ activeNode?.name }}</p>
        <p class="text-xs text-slate-400 max-w-[220px] leading-relaxed mt-1">
          Cette entité n'a aucun lien direct répertorié dans le sous-graphe actuel.
        </p>
      </div>

      <div v-else-if="!activeNode" class="text-center py-16 text-slate-400 flex flex-col items-center justify-center min-h-[250px]">
        <span class="text-2xl mb-2">🕵️‍♂️</span>
        <p class="text-xs font-medium max-w-[200px] leading-relaxed">
          En attente de cible. Cliquez sur une entité du diagramme principal pour cartographier ses relations au 1er degré.
        </p>
      </div>

      <div v-else>
        <div class="text-[9px] font-bold text-blue-600 uppercase tracking-wider mb-2 text-left px-2 flex justify-between items-center">
          <span>🎯 Cible active : {{ activeNode.name }}</span>
          <span class="text-slate-400 font-normal">Voisinage (1er degré)</span>
        </div>
        <svg ref="svgRef" class="w-full bg-slate-50/60 rounded-xl border border-slate-100 min-h-[250px]"></svg>
      </div>
    </div>
  </div>
</template>