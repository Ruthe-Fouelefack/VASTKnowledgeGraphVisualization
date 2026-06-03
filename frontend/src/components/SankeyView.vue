<script setup>
import { onMounted, ref } from 'vue'
import * as d3 from 'd3'
import { sankey as d3Sankey, sankeyLinkHorizontal } from 'd3-sankey'

const loading = ref(true)
const svgRef = ref(null)
const debugMessage = ref("")

onMounted(async () => {
  try {
    const response = await fetch('/MC1_graph.json')
    const data = await response.json()
    
    if (!data || !data.nodes || !data.links) {
      debugMessage.value = "Fichier JSON introuvable ou corrompu."
      loading.value = false
      return
    }

    // 1. Indexer tous les types de nœuds existants
    const nodeTypeMap = new Map()
    data.nodes.forEach(n => {
      if (n && n.id !== undefined) {
        nodeTypeMap.set(String(n.id), n['Node Type'] || 'Inconnu')
      }
    })

    // 2. Calculer les flux agrégés (uniquement pour les liens valides)
    const flows = {}
    const rawLinks = data.links || data.edges || []

    rawLinks.forEach(l => {
      const srcId = String(l.source?.id !== undefined ? l.source.id : l.source)
      const tgtId = String(l.target?.id !== undefined ? l.target.id : l.target)
      
      if (nodeTypeMap.has(srcId) && nodeTypeMap.has(tgtId)) {
        const srcType = nodeTypeMap.get(srcId)
        const tgtType = nodeTypeMap.get(tgtId)
        const edgeType = l['Edge Type'] || l.type || 'Lien'

        const key1 = `S_${srcType}➔E_${edgeType}`
        flows[key1] = (flows[key1] || 0) + 1

        const key2 = `E_${edgeType}➔T_${tgtType}`
        flows[key2] = (flows[key2] || 0) + 1
      }
    })

    // 3. Convertir pour l'algorithme Sankey
    const nodesMap = new Map()
    let idx = 0
    const getNodeIdx = (name, layer) => {
      const key = `${layer}_${name}`
      if (!nodesMap.has(key)) {
        nodesMap.set(key, { index: idx++, name, layer })
      }
      return nodesMap.get(key).index
    }

    const sankeyLinks = []
    Object.entries(flows).forEach(([key, value]) => {
      const [part1, part2] = key.split('➔')
      const srcLayer = part1.startsWith('S_') ? 0 : 1
      const srcName = part1.substring(2)
      const tgtLayer = part2.startsWith('E_') ? 1 : 2
      const tgtName = part2.substring(2)

      const srcIdx = getNodeIdx(srcName, srcLayer)
      const tgtIdx = getNodeIdx(tgtName, tgtLayer)

      sankeyLinks.push({ source: srcIdx, target: tgtIdx, value })
    })

    const sankeyNodes = Array.from(nodesMap.values()).sort((a, b) => a.index - b.index)

    if (sankeyLinks.length === 0) {
      debugMessage.value = "Aucun flux valide généré."
      loading.value = false
      return
    }

    // 4. Configuration graphique fixe
    const width = 600
    const height = 300

    // Ciblage et nettoyage complet du SVG
    const svg = d3.select(svgRef.value)
      .attr('viewBox', `0 0 ${width} ${height}`)
      .attr('width', '100%')
      .attr('height', height)
    
    svg.selectAll('*').remove()

    // Configuration de d3-sankey
    const sankeyGenerator = d3Sankey()
      .nodeWidth(16)
      .nodePadding(14)
      .extent([[20, 20], [width - 20, height - 20]])

    const { nodes, links } = sankeyGenerator({
      nodes: sankeyNodes.map(d => ({ ...d })),
      links: sankeyLinks.map(d => ({ ...d }))
    })

    // Palette de couleurs
    const colorNode = (d) => {
      if (d.layer === 1) return '#94a3b8' // Relations en gris
      switch(d.name) {
        case 'Person': return '#3b82f6'      // Bleu
        case 'Song': return '#10b981'        // Vert
        case 'RecordLabel': return '#ef4444' // Rouge
        default: return '#64748b'
      }
    }

    // Main group conteneur global
    const mainGroup = svg.append('g').attr('class', 'sankey-main')

    // 5. Rendu des rubans (Liens)
    mainGroup.append('g')
      .attr('fill', 'none')
      .attr('stroke-opacity', 0.25)
      .selectAll('path')
      .data(links)
      .join('path')
      .attr('d', sankeyLinkHorizontal())
      .attr('stroke', d => colorNode(d.source))
      .attr('stroke-width', d => Math.max(2, d.width))
      .append('title')
      .text(d => `${d.source.name} ➔ ${d.target.name}\n${d.value.toLocaleString()} relations`)

    // 6. Rendu des rectangles (Nœuds)
    const node = mainGroup.append('g')
      .selectAll('g')
      .data(nodes)
      .join('g')

    node.append('rect')
      .attr('x', d => d.x0)
      .attr('y', d => d.y0)
      .attr('height', d => d.y1 - d.y0)
      .attr('width', d => d.x1 - d.x0)
      .attr('fill', d => colorNode(d))
      .attr('rx', 3)

    // 7. Textes
    node.append('text')
      .attr('x', d => d.x0 < width / 2 ? d.x1 + 6 : d.x0 - 6)
      .attr('y', d => (d.y0 + d.y1) / 2)
      .attr('dy', '0.35em')
      .attr('text-anchor', d => d.x0 < width / 2 ? 'start' : 'end')
      .attr('font-family', 'sans-serif')
      .attr('font-size', '10px')
      .attr('font-weight', '700')
      .attr('fill', '#334155')
      .text(d => d.name)

    loading.value = false

  } catch (error) {
    console.error("❌ Erreur de rendu D3 Sankey :", error)
    debugMessage.value = error.message
    loading.value = false
  }
})
</script>

<template>
  <div class="p-2 bg-white rounded text-center">
    <p v-if="loading" class="text-xs text-amber-600 animate-pulse py-16">
      Calcul et alignement de la matrice des flux...
    </p>
    <div v-else-if="debugMessage" class="text-xs text-red-500 py-16 font-mono">
      ⚠️ {{ debugMessage }}
    </div>

    <div :class="{ 'hidden': loading || debugMessage }">
      <div class="text-[9px] font-bold text-slate-400 uppercase tracking-wider mb-3 text-left px-2">
        ENTITÉS ÉMETTRICES ➔ TYPES DE RELATIONS ➔ DESTINATAIRES
      </div>
      <svg ref="svgRef" class="w-full bg-slate-50/50 rounded-xl border border-slate-100 min-h-[300px]"></svg>
    </div>
  </div>
</template>