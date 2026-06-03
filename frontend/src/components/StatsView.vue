<script setup>
import { computed } from 'vue'

// On reçoit les données filtrées actuelles depuis le composant parent
const props = defineProps({
  activeNodes: {
    type: Array,
    default: () => []
  }
})

// Calcul dynamique du nombre d'éléments par type
const stats = computed(() => {
  const counts = { Person: 0, Song: 0, RecordLabel: 0, Other: 0 }
  
  props.activeNodes.forEach(node => {
    const type = node['Node Type']
    if (counts[type] !== undefined) {
      counts[type]++
    } else {
      counts[type]['Other']++
    }
  })
  
  return counts
})

// Calcul du total pour les pourcentages
const total = computed(() => props.activeNodes.length)
</script>

<template>
  <div class="p-4 bg-white rounded-lg h-full">
    <h3 class="text-sm font-bold text-slate-700 mb-4">Répartition des entités visibles</h3>
    
    <div v-if="total === 0" class="text-center py-12 text-xs text-slate-400 font-medium">
      Aucune donnée à analyser. Cochez des filtres !
    </div>
    
    <div v-else class="space-y-4">
      <div class="bg-slate-50 p-3 rounded-lg border border-slate-100 text-center">
        <p class="text-[10px] uppercase tracking-wider font-semibold text-slate-400">Total affiché</p>
        <p class="text-2xl font-black text-slate-800">{{ total }}</p>
      </div>

      <div class="space-y-1">
        <div class="flex justify-between text-xs font-medium text-slate-600">
          <span class="flex items-center gap-1.5">
            <span class="w-2.5 h-2.5 rounded-full bg-blue-500"></span> Personnes
          </span>
          <span>{{ stats.Person }} ({{ total ? Math.round((stats.Person / total) * 100) : 0 }}%)</span>
        </div>
        <div class="w-full bg-slate-100 rounded-full h-2">
          <div class="bg-blue-500 h-2 rounded-full transition-all duration-500" :style="{ width: `${total ? (stats.Person / total) * 100 : 0}%` }"></div>
        </div>
      </div>

      <div class="space-y-1">
        <div class="flex justify-between text-xs font-medium text-slate-600">
          <span class="flex items-center gap-1.5">
            <span class="w-2.5 h-2.5 rounded-full bg-emerald-500"></span> Chansons
          </span>
          <span>{{ stats.Song }} ({{ total ? Math.round((stats.Song / total) * 100) : 0 }}%)</span>
        </div>
        <div class="w-full bg-slate-100 rounded-full h-2">
          <div class="bg-emerald-500 h-2 rounded-full transition-all duration-500" :style="{ width: `${total ? (stats.Song / total) * 100 : 0}%` }"></div>
        </div>
      </div>

      <div class="space-y-1">
        <div class="flex justify-between text-xs font-medium text-slate-600">
          <span class="flex items-center gap-1.5">
            <span class="w-2.5 h-2.5 rounded-full bg-red-500"></span> Labels
          </span>
          <span>{{ stats.RecordLabel }} ({{ total ? Math.round((stats.RecordLabel / total) * 100) : 0 }}%)</span>
        </div>
        <div class="w-full bg-slate-100 rounded-full h-2">
          <div class="bg-red-500 h-2 rounded-full transition-all duration-500" :style="{ width: `${total ? (stats.RecordLabel / total) * 100 : 0}%` }"></div>
        </div>
      </div>
    </div>
  </div>
</template>