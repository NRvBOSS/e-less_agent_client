<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'
import axios from 'axios'
import Dashboard from './Dashboard.vue'

interface Log {
  id?: number | string
  server: string
  message: string
  analysis: string
  time: string
}

const logs = ref<Log[]>([])
const loading = ref(true)
const lastUpdated = ref<Date | null>(null)
let pollTimer: ReturnType<typeof setInterval> | null = null

async function loadLogs() {
  try {
    const res = await axios.get<Log[]>('http://localhost:8080/analysis')
    logs.value = res.data.map((l, i) => ({ ...l, id: l.id ?? i }))
    lastUpdated.value = new Date()
  } catch (e) {
    console.error('Failed to load logs', e)
  } finally {
    loading.value = false
  }
}

onMounted(() => {
  loadLogs()
  pollTimer = setInterval(loadLogs, 15_000)
})
onUnmounted(() => {
  if (pollTimer) clearInterval(pollTimer)
})
</script>

<template>
  <Dashboard
    :logs="logs"
    :loading="loading"
    :last-updated="lastUpdated"
    @refresh="loadLogs"
  />
</template>