<script setup>
import { onMounted, ref, computed, watch } from 'vue'
import * as d3 from 'd3'

// --- ÉVÉNEMENTS AUTORISÉS (DÉCLARATION COMBINÉE) ---
// Émet le nœud cliqué vers le composant d'orchestration parent pour synchroniser l'Ego Network
const emit = defineEmits(['node-click'])

// --- ÉTATS RÉACTIFS VUE ---
const loading = ref(true)
const svgRef = ref(null)

// Données sources stables (clonées au départ)
const allNodes = ref([])
const allLinks = ref([])

// États de filtrage interactifs (Sidebar embarquée / Contrôles)
const searchQuery = ref('')
const filterPerson = ref(true)
const filterSong = ref(true)
const filterRecordLabel = ref(true)

// Élément sélectionné au clic (Inspecteur de preuves de droite)
const selectedEntity = ref(null)

// Variables D3 globales pour la simulation cinétique
let simulation = null
let linkSelection = null
let nodeSelection = null
let labelSelection = null

// --- ALGORITHME DE FILTRAGE SÉCURISÉ ---
const filteredData = computed(() => {
  if (!allNodes.value.length) return { nodes: [], links: [] }

  const query = searchQuery.value.trim().toLowerCase()

  // 1. Étape 1 : Filtrage de base selon les Checkboxes (Dimensions T Nodes)
  let baseNodes = allNodes.value.filter(n => {
    if (n['Node Type'] === 'Person' && !filterPerson.value) return false
    if (n['Node Type'] === 'Song' && !filterSong.value) return false
    if (n['Node Type'] === 'RecordLabel' && !filterRecordLabel.value) return false
    return true
  })

  // 2. Étape 2 : Si une recherche textuelle est active (Focus sur un artiste)
  if (query !== '') {
    const searchedNodes = baseNodes.filter(n => n.name && n.name.toLowerCase().includes(query))
    const searchedIds = new Set(searchedNodes.map(n => n.id))

    const neighborIds = new Set()
    const connectedLinks = allLinks.value.filter(link => {
      const sId = String(link.source && link.source.id ? link.source.id : link.source)
      const tId = String(link.target && link.target.id ? link.target.id : link.target)
      
      if (searchedIds.has(sId)) { neighborIds.add(tId); return true }
      if (searchedIds.has(tId)) { neighborIds.add(sId); return true }
      return false
    })

    const finalIds = new Set([...searchedIds, ...neighborIds])
    const finalNodes = baseNodes.filter(n => finalIds.has(n.id))

    return { nodes: finalNodes, links: connectedLinks }
  }

  // 3. Étape 3 : Si pas de recherche active, affichage d'un échantillon topologique stable
  const defaultNodes = baseNodes.slice(0, 250)
  const defaultIds = new Set(defaultNodes.map(n => n.id))
  
  const defaultLinks = allLinks.value.filter(link => {
    const sId = String(link.source && link.source.id ? link.source.id : link.source)
    const tId = String(link.target && link.target.id ? link.target.id : link.target)
    return defaultIds.has(sId) && defaultIds.has(tId)
  })

  return { nodes: defaultNodes, links: defaultLinks }
})

// --- FONCTION DE MISE À JOUR DE LA SIMULATION (DIAGRAMME FORCE-DIRECTED) ---
const updateGraph = () => {
  if (!svgRef.value || !simulation) return

  const { nodes, links } = filteredData.value

  const svg = d3.select(svgRef.value)
  let g = svg.select('g.main-group')
  if (g.empty()) g = svg.append('g').attr('class', 'main-group')

  // Charte chromatique sémantique unifiée (Gestalt : Loi de Similarité)
  const getColor = (type) => {
    switch(type) {
      case 'Person': return '#3b82f6'       // Bleu
      case 'Song': return '#10b981'         // Vert
      case 'RecordLabel': return '#ef4444'  // Rouge
      default: return '#94a3b8'
    }
  }

  // Injection des structures filtrées dans le moteur physique D3
  simulation.nodes(nodes)
  simulation.force('link').links(links)
  simulation.alpha(0.3).restart()

  // Rendu et jointure des lignes (T Links)
  linkSelection = g.selectAll('line')
    .data(links, d => {
      const s = d.source.id || d.source
      const t = d.target.id || d.target
      return `${s}-${t}`
    })
    .join(
      enter => enter.append('line').attr('stroke', '#cbd5e1').attr('stroke-opacity', 0.6).attr('stroke-width', 1.5),
      update => update,
      exit => exit.remove()
    )

  // Configuration interactive du Drag & Drop pour la manipulation des nœuds
  const drag = (sim) => {
    function dragstarted(event, d) {
      if (!event.active) sim.alphaTarget(0.2).restart()
      d.fx = d.x; d.fy = d.y
    }
    function dragged(event, d) { d.fx = event.x; d.fy = event.y }
    function dragended(event, d) {
      if (!event.active) sim.alphaTarget(0)
      d.fx = null; d.fy = null
    }
    return d3.drag().on('start', dragstarted).on('drag', dragged).on('end', dragended)
  }

  // Rendu et jointure des cercles (T Nodes)
  nodeSelection = g.selectAll('circle')
    .data(nodes, d => d.id)
    .join(
      enter => {
        const circle = enter.append('circle')
          .attr('r', 8)
          .attr('fill', d => getColor(d['Node Type']))
          .attr('stroke', '#ffffff')
          .attr('stroke-width', 1.5)
          .style('cursor', 'grab')
          .call(drag(simulation))

        // Interconnexion réactive : Clic sur le diagramme principal
        circle.on('click', (event, d) => { 
          selectedEntity.value = d        // Alimente l'inspecteur de métadonnées local
          emit('node-click', d)           // Émet vers HomeView pour recalculer l'Ego Network
        })
        
        circle.append('title').text(d => `${d.name} (${d['Node Type']})`)
        return circle
      },
      update => update.attr('fill', d => getColor(d['Node Type'])),
      exit => exit.remove()
    )

  // Rendu et jointure des étiquettes textuelles (Labels)
  labelSelection = g.selectAll('text')
    .data(nodes, d => d.id)
    .join(
      enter => enter.append('text')
        .attr('dy', -12)
        .attr('text-anchor', 'middle')
        .attr('font-size', '9px')
        .attr('font-weight', '500')
        .attr('fill', '#475569')
        .text(d => d.name && d.name.length > 12 ? d.name.substring(0, 10) + '...' : d.name),
      update => update.text(d => d.name && d.name.length > 12 ? d.name.substring(0, 10) + '...' : d.name),
      exit => exit.remove()
    )
}

onMounted(async () => {
  try {
    // Ingestion initiale du graphe de connaissances
    const response = await fetch('/MC1_graph.json')
    const data = await response.json()
    
    allNodes.value = (data.nodes || []).map(d => ({ ...d, id: String(d.id) }))
    allLinks.value = (data.links || data.edges || []).map(d => ({
      source: String(d.source),
      target: String(d.target),
      type: d['Edge Type']
    }))
    
    loading.value = false

    const width = 700
    const height = 450
    const svg = d3.select(svgRef.value)
      .attr('width', '100%').attr('height', height).attr('viewBox', [0, 0, width, height])
    
    // Ajout des fonctionnalités globales de Zoom et Pan interactif
    svg.call(d3.zoom().scaleExtent([0.1, 8]).on('zoom', (event) => {
      svg.select('g.main-group').attr('transform', event.transform)
    }))

    // Configuration des contraintes et forces du modèle de réseau spatialisé
    simulation = d3.forceSimulation()
      .force('link', d3.forceLink().id(d => d.id).distance(55))
      .force('charge', d3.forceManyBody().strength(-90))
      .force('center', d3.forceCenter(width / 2, height / 2))
      .force('x', d3.forceX(width / 2).strength(0.12))
      .force('y', d3.forceY(height / 2).strength(0.12))

    // Rafraîchissement cinétique des coordonnées à chaque itération physique (Tick)
    simulation.on('tick', () => {
      if (linkSelection) {
        linkSelection.attr('x1', d => d.source.x).attr('y1', d => d.source.y)
                     .attr('x2', d => d.target.x).attr('y2', d => d.target.y)
      }
      if (nodeSelection) nodeSelection.attr('cx', d => d.x).attr('cy', d => d.y)
      if (labelSelection) labelSelection.attr('x', d => d.x).attr('y', d => d.y)
    })

    updateGraph()

    // Boucle de rétroaction : écoute des modifications de filtres pour reconstruire le réseau
    watch([filterPerson, filterSong, filterRecordLabel, searchQuery], () => {
      updateGraph()
    })

  } catch (error) {
    console.error("❌ Erreur d'initialisation de l'enquête :", error)
  }
})

// Variables calculées pour le panneau de statistiques (Filter Feedback Loop)
const countPersons = computed(() => filteredData.value.nodes.filter(n => n['Node Type'] === 'Person').length)
const countSongs = computed(() => filteredData.value.nodes.filter(n => n['Node Type'] === 'Song').length)
const countLabels = computed(() => filteredData.value.nodes.filter(n => n['Node Type'] === 'RecordLabel').length)
</script>

<template>
  <div class="flex flex-col lg:flex-row gap-6 p-4 bg-slate-100 min-h-screen font-sans">
    
    <div class="flex-1 bg-white p-5 rounded-2xl shadow-sm border border-slate-200">
      <div class="mb-2">
        <h2 class="text-lg font-bold text-slate-800">🕵️‍♂️ Enquête Oceanus Folk : Then-and-Now</h2>
        <p class="text-xs text-slate-400">Outil d'analyse visuelle pour le journaliste Silas Reed (Période 2023 - 2040)</p>
      </div>

      <div class="flex flex-wrap items-center justify-between gap-4 mb-5 border-b border-slate-100 pb-4">
        <div class="flex items-center gap-2">
          <input 
            v-model="searchQuery"
            type="text" 
            placeholder="🔍 Chercher Sailor Shift, Maya, Lilly..." 
            class="px-4 py-2 text-sm border border-slate-200 rounded-lg bg-slate-50 focus:outline-none focus:ring-2 focus:ring-blue-500 w-72 text-slate-700"
          />
          <button 
            @click="searchQuery = 'Sailor Shift'"
            type="button"
            class="text-[10px] bg-blue-50 border border-blue-200 text-blue-600 font-semibold px-2 py-1 rounded hover:bg-blue-100 transition"
          >
            🔍 Focus Sailor
          </button>
          <button 
            @click="searchQuery = ''"
            type="button"
            class="text-[10px] bg-slate-100 text-slate-500 px-2 py-1 rounded hover:bg-slate-200 transition"
            v-if="searchQuery"
          >
            Réinitialiser
          </button>
        </div>
        
        <div class="flex items-center gap-4 text-xs font-semibold text-slate-600">
          <label class="flex items-center gap-1.5 cursor-pointer select-none">
            <input type="checkbox" v-model="filterPerson" class="rounded text-blue-500 w-4 h-4" />
            <span class="w-2.5 h-2.5 rounded-full bg-blue-500"></span> Personnes
          </label>
          <label class="flex items-center gap-1.5 cursor-pointer select-none">
            <input type="checkbox" v-model="filterSong" class="rounded text-emerald-500 w-4 h-4" />
            <span class="w-2.5 h-2.5 rounded-full bg-emerald-500"></span> Chansons
          </label>
          <label class="flex items-center gap-1.5 cursor-pointer select-none">
            <input type="checkbox" v-model="filterRecordLabel" class="rounded text-red-500 w-4 h-4" />
            <span class="w-2.5 h-2.5 rounded-full bg-red-500"></span> Labels
          </label>
        </div>
      </div>

      <div class="relative">
        <p v-if="loading" class="text-xs text-amber-600 animate-pulse text-center py-24">
          Chargement de l'univers musical d'Oceanus...
        </p>
        <svg ref="svgRef" class="bg-slate-50 rounded-xl border border-slate-100 shadow-inner w-full cursor-move"></svg>
        <p class="text-[10px] text-slate-400 mt-2 text-right">💡 Utilisez la molette pour zoomer, glissez le fond pour vous déplacer et déplacez les points à la souris.</p>
      </div>
    </div>

    <div class="w-full lg:w-80 flex flex-col gap-6">
      
      <div class="bg-white p-5 rounded-2xl shadow-sm border border-slate-200">
        <h3 class="text-sm font-bold text-slate-800 mb-4 flex items-center gap-2">📊 Éléments Filtrés</h3>
        <div class="space-y-3 text-xs">
          <div>
            <div class="flex justify-between text-slate-500 mb-1">
              <span>Personnes</span> <span class="font-bold text-blue-600">{{ countPersons }}</span>
            </div>
            <div class="w-full bg-slate-100 h-1.5 rounded-full overflow-hidden">
              <div class="bg-blue-500 h-full transition-all duration-300" :style="{ width: (countPersons * 100 / (filteredData.nodes.length || 1)) + '%' }"></div>
            </div>
          </div>
          <div>
            <div class="flex justify-between text-slate-500 mb-1">
              <span>Chansons</span> <span class="font-bold text-emerald-600">{{ countSongs }}</span>
            </div>
            <div class="w-full bg-slate-100 h-1.5 rounded-full overflow-hidden">
              <div class="bg-emerald-500 h-full transition-all duration-300" :style="{ width: (countSongs * 100 / (filteredData.nodes.length || 1)) + '%' }"></div>
            </div>
          </div>
          <div>
            <div class="flex justify-between text-slate-500 mb-1">
              <span>Labels de Disques</span> <span class="font-bold text-red-600">{{ countLabels }}</span>
            </div>
            <div class="w-full bg-slate-100 h-1.5 rounded-full overflow-hidden">
              <div class="bg-red-500 h-full transition-all duration-300" :style="{ width: (countLabels * 100 / (filteredData.nodes.length || 1)) + '%' }"></div>
            </div>
          </div>
          <p class="text-[10px] text-slate-400 border-t border-slate-100 pt-2 mt-2">
            Réseau actif : {{ filteredData.nodes.length }} entités, {{ filteredData.links.length }} relations.
          </p>
        </div>
      </div>

      <div class="bg-white p-5 rounded-2xl shadow-sm border border-slate-200 flex-1 min-h-[250px]">
        <h3 class="text-sm font-bold text-slate-800 mb-3 flex items-center gap-2">🔍 Preuves & Métadonnées</h3>
        
        <div v-if="selectedEntity" class="space-y-3">
          <div class="bg-slate-50 p-3 rounded-xl border border-slate-100">
            <p class="text-[10px] uppercase tracking-wider text-slate-400 font-bold">Nom de l'entité</p>
            <p class="text-sm font-bold text-slate-800">{{ selectedEntity.name }}</p>
          </div>

          <div class="grid grid-cols-2 gap-3">
            <div class="bg-slate-50 p-2.5 rounded-xl border border-slate-100">
              <p class="text-[10px] uppercase tracking-wider text-slate-400 font-bold">Catégorie</p>
              <span class="text-xs font-semibold px-2 py-0.5 rounded-full inline-block mt-1"
                :class="{
                  'bg-blue-100 text-blue-700': selectedEntity['Node Type'] === 'Person',
                  'bg-emerald-100 text-emerald-700': selectedEntity['Node Type'] === 'Song',
                  'bg-red-100 text-red-700': selectedEntity['Node Type'] === 'RecordLabel'
                }">
                {{ selectedEntity['Node Type'] }}
              </span>
            </div>
            <div class="bg-slate-50 p-2.5 rounded-xl border border-slate-100">
              <p class="text-[10px] uppercase tracking-wider text-slate-400 font-bold">ID Réseau</p>
              <p class="text-xs font-mono text-slate-600 mt-1">#{{ selectedEntity.id }}</p>
            </div>
          </div>

          <div class="bg-slate-50 p-3 rounded-xl border border-slate-100 text-xs text-slate-600 space-y-2">
            <p class="font-bold border-b border-slate-200 pb-1 mb-1 text-slate-700">📋 Attributs d'Enquête :</p>
            <p v-if="selectedEntity.genre">🎵 <strong>Genre Musical :</strong> {{ selectedEntity.genre }}</p>
            <p v-if="selectedEntity.release_date">📅 <strong>Année de Sortie :</strong> {{ selectedEntity.release_date }}</p>
            <p v-if="selectedEntity.single !== undefined">💿 <strong>Single Officiel :</strong> {{ selectedEntity.single ? 'Oui' : 'Non' }}</p>
          </div>
        </div>

        <div v-else class="text-center py-12 text-slate-400 flex flex-col items-center justify-center h-full">
          <span class="text-2xl mb-1">🕵️‍♂️</span>
          <p class="text-xs max-w-[180px]">Cliquez sur un suspect ou un morceau pour retracer son parcours d'influence et peupler l'Ego Network.</p>
        </div>
      </div>

    </div>
  </div>
</template>