<script setup>
import { ref, onMounted } from "vue"
import axios from "axios"

const logs = ref([])

async function loadLogs() {
  const res = await axios.get("http://localhost:8080/analysis")
  logs.value = res.data
}

onMounted(() => {
  loadLogs()
})
</script>

<template>
  <div style="padding:20px">
    <h1>LogMind Dashboard</h1>

    <table border="1" cellpadding="10">
      <thead>
        <tr>
          <th>Server</th>
          <th>Error</th>
          <th>AI Solution</th>
          <th>Time</th>
        </tr>
      </thead>

      <tbody>
        <tr v-for="log in logs" :key="log.time">
          <td>{{ log.server }}</td>
          <td style="max-width:400px">{{ log.message }}</td>
          <td style="max-width:400px">{{ log.analysis }}</td>
          <td>{{ log.time }}</td>
        </tr>
      </tbody>
    </table>
  </div>
</template>