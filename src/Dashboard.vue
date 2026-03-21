<script setup lang="ts">
import { ref, computed } from 'vue'

type Severity = 'CRITICAL' | 'ERROR' | 'WARN' | 'INFO' | 'DEBUG'
type SevFilter = 'ALL' | Severity

interface Log {
  id?: number | string
  server: string
  message: string
  analysis: string
  time: string
  severity?: Severity
}

const props = defineProps<{
  logs: Log[]
  loading: boolean
  lastUpdated: Date | null
}>()

defineEmits<{ refresh: [] }>()

// ── State
const search   = ref('')
const sevFilter = ref<SevFilter>('ALL')
const expanded  = ref<number | string | null>(null)

// ── Helpers
const SEVS: Severity[] = ['CRITICAL', 'ERROR', 'WARN', 'INFO', 'DEBUG']
const ALL_FILTERS: SevFilter[] = ['ALL', ...SEVS]

function inferSeverity(msg = ''): Severity {
  const m = msg.toUpperCase()
  if (m.includes('CRITICAL') || m.includes('FATAL') || m.includes('PANIC')) return 'CRITICAL'
  if (m.includes('ERROR') || m.includes('ERR'))  return 'ERROR'
  if (m.includes('WARN'))  return 'WARN'
  if (m.includes('INFO'))  return 'INFO'
  return 'DEBUG'
}

const enriched = computed(() =>
  props.logs.map((l, i) => ({
    ...l,
    id: l.id ?? i,
    severity: l.severity ?? inferSeverity(l.message),
  }))
)

const counts = computed(() => {
  const c = { CRITICAL: 0, ERROR: 0, WARN: 0, INFO: 0, DEBUG: 0 }
  enriched.value.forEach(l => { c[l.severity]++ })
  return c
})

const filtered = computed(() => {
  const q = search.value.toLowerCase()
  return enriched.value.filter(l => {
    const matchSev = sevFilter.value === 'ALL' || l.severity === sevFilter.value
    const matchQ   = !q || l.server?.toLowerCase().includes(q) ||
                     l.message?.toLowerCase().includes(q) || l.analysis?.toLowerCase().includes(q)
    return matchSev && matchQ
  })
})

function toggle(id: number | string) {
  expanded.value = expanded.value === id ? null : id
}

function fmtTime(ts: string | Date | null | undefined) {
  if (!ts) return '—'
  const d = new Date(ts)
  if (isNaN(d.getTime())) return String(ts)
  return d.toLocaleString('en-US', { month: '2-digit', day: '2-digit', hour: '2-digit', minute: '2-digit', second: '2-digit', hour12: false })
}

function fmtRel(ts: string | Date | null | undefined) {
  if (!ts) return ''
  const s = (Date.now() - new Date(ts).getTime()) / 1000
  if (s < 60)   return `${Math.floor(s)}s ago`
  if (s < 3600) return `${Math.floor(s / 60)}m ago`
  return `${Math.floor(s / 3600)}h ago`
}

const SEV_STYLE: Record<Severity, { badge: string; leftBar: string; pillActive: string }> = {
  CRITICAL: { badge: 'text-red-400 bg-red-500/10 border-red-500/30',    leftBar: 'border-l-red-500',    pillActive: 'bg-red-500/10 border-red-500 text-red-400'    },
  ERROR:    { badge: 'text-orange-400 bg-orange-500/10 border-orange-500/30', leftBar: 'border-l-orange-500', pillActive: 'bg-orange-500/10 border-orange-500 text-orange-400' },
  WARN:     { badge: 'text-yellow-400 bg-yellow-500/10 border-yellow-500/30', leftBar: 'border-l-yellow-500', pillActive: 'bg-yellow-500/10 border-yellow-500 text-yellow-400' },
  INFO:     { badge: 'text-sky-400 bg-sky-500/10 border-sky-500/30',     leftBar: 'border-l-sky-500',    pillActive: 'bg-sky-500/10 border-sky-500 text-sky-400'     },
  DEBUG:    { badge: 'text-slate-400 bg-slate-500/10 border-slate-500/20', leftBar: 'border-l-slate-600',  pillActive: 'bg-slate-500/10 border-slate-500 text-slate-400' },
}
</script>

<template>
  <div class="flex flex-col h-screen bg-[#0d0f14] text-slate-200 overflow-hidden font-sans">

    <!-- ── HEADER ──────────────────────────────────────────────────────────── -->
    <header class="shrink-0 flex items-center justify-between h-12 px-6 bg-[#13161e] border-b border-[#222636]">
      <div class="flex items-center gap-2">
        <span class="text-violet-500 text-xl leading-none">⬡</span>
        <span class="text-[15px] font-semibold tracking-widest text-slate-100">E-Less</span>
        <span class="font-mono text-xs text-slate-500">/ dashboard</span>
      </div>
      <div class="flex items-center gap-3">
        <span class="w-2 h-2 rounded-full bg-emerald-400 shadow-[0_0_6px_#34d399] animate-pulse" />
        <span class="font-mono text-[10px] font-semibold tracking-widest text-emerald-400">LIVE</span>
        <span v-if="lastUpdated" class="font-mono text-[11px] text-slate-500 hidden sm:block">
          Updated {{ fmtRel(lastUpdated) }}
        </span>
        <button
          class="w-7 h-7 flex items-center justify-center rounded border border-[#2e3450]
                 text-slate-400 hover:text-white hover:border-violet-500 transition-all bg-transparent cursor-pointer"
          title="Refresh"
          @click="$emit('refresh')"
        >↺</button>
      </div>
    </header>

    <!-- ── STAT PILLS ──────────────────────────────────────────────────────── -->
    <div class="shrink-0 flex flex-wrap items-center gap-2 px-6 py-3 border-b border-[#222636]">
      <button
        v-for="sev in SEVS" :key="sev"
        class="flex items-center gap-1.5 px-3 py-1 rounded-full border font-mono text-xs cursor-pointer transition-all"
        :class="sevFilter === sev
          ? SEV_STYLE[sev].pillActive + ' border'
          : 'bg-[#13161e] border-[#222636] text-slate-500 hover:border-[#2e3450]'"
        @click="sevFilter = sevFilter === sev ? 'ALL' : sev"
      >
        <span class="text-sm font-bold leading-none">{{ counts[sev] }}</span>
        <span class="text-[9px] tracking-widest">{{ sev }}</span>
      </button>
      <div class="flex items-center gap-1.5 px-3 py-1 font-mono">
        <span class="text-sm font-bold text-slate-200 leading-none">{{ logs.length }}</span>
        <span class="text-[9px] tracking-widest text-slate-500">TOTAL</span>
      </div>
    </div>

    <!-- ── TOOLBAR ─────────────────────────────────────────────────────────── -->
    <div class="shrink-0 flex flex-wrap items-center gap-3 px-6 py-3 border-b border-[#222636]">
      <!-- Search -->
      <div class="relative flex-1 min-w-[200px]">
        <span class="absolute left-3 top-1/2 -translate-y-1/2 text-slate-500 text-base pointer-events-none select-none">⌕</span>
        <input
          v-model="search"
          class="w-full bg-[#0d0f14] border border-[#222636] rounded-md text-slate-200
                 font-mono text-xs py-2 pl-8 pr-7 outline-none
                 focus:border-violet-500 placeholder:text-slate-600 transition-colors"
          placeholder="Search server, message, analysis…"
          spellcheck="false"
        />
        <button
          v-if="search"
          class="absolute right-2 top-1/2 -translate-y-1/2 text-slate-500 hover:text-slate-300
                 bg-transparent border-none cursor-pointer text-xs"
          @click="search = ''"
        >✕</button>
      </div>
      <!-- Filter tabs -->
      <div class="flex flex-wrap gap-1">
        <button
          v-for="s in ALL_FILTERS" :key="s"
          class="px-3 py-1 rounded border font-mono text-[10px] tracking-widest cursor-pointer transition-all"
          :class="sevFilter === s
            ? 'bg-violet-600 border-violet-600 text-white'
            : 'bg-[#13161e] border-[#222636] text-slate-500 hover:border-[#2e3450] hover:text-slate-300'"
          @click="sevFilter = s"
        >{{ s }}</button>
      </div>
    </div>

    <!-- ── TABLE AREA (scrollable) ─────────────────────────────────────────── -->
    <div class="flex-1 min-h-0 overflow-auto px-6 py-4">

      <!-- Count -->
      <p class="font-mono text-[11px] text-slate-500 mb-3">
        Showing <span class="text-slate-200 font-semibold">{{ filtered.length }}</span> of {{ logs.length }} events
      </p>

      <!-- Card border wrapping the table -->
      <div class="rounded-lg border border-[#222636] overflow-hidden">

        <!-- Skeleton -->
        <div v-if="loading">
          <div v-for="n in 6" :key="n" class="flex gap-4 px-4 py-3 border-b border-[#222636] last:border-b-0 animate-pulse">
            <div class="h-4 w-16 rounded bg-[#1e2232]" />
            <div class="h-4 w-28 rounded bg-[#1e2232]" />
            <div class="h-4 flex-1 rounded bg-[#1e2232]" />
            <div class="h-4 w-20 rounded bg-[#1e2232]" />
          </div>
        </div>

        <!-- Empty -->
        <div v-else-if="!filtered.length" class="flex flex-col items-center justify-center py-20 gap-3">
          <span class="text-4xl text-slate-700">◎</span>
          <p class="text-slate-500 font-mono text-sm">No events match your filters.</p>
          <button
            class="mt-1 px-4 py-2 text-sm rounded border border-[#2e3450] text-slate-300
                   bg-[#13161e] hover:border-violet-500 transition-colors cursor-pointer"
            @click="search = ''; sevFilter = 'ALL'"
          >Clear filters</button>
        </div>

        <!-- Table -->
        <table v-else class="w-full border-collapse text-[13px]" style="table-layout:fixed">
          <colgroup>
            <col style="width:100px" />
            <col style="width:145px" />
            <col style="width:28%" />
            <col />
            <col style="width:135px" />
          </colgroup>
          <thead>
            <tr class="bg-[#13161e] border-b border-[#222636]">
              <th class="px-4 py-2.5 text-left font-mono text-[10px] font-semibold tracking-widest text-slate-500">SEV</th>
              <th class="px-4 py-2.5 text-left font-mono text-[10px] font-semibold tracking-widest text-slate-500">SERVER</th>
              <th class="px-4 py-2.5 text-left font-mono text-[10px] font-semibold tracking-widest text-slate-500">ERROR MESSAGE</th>
              <th class="px-4 py-2.5 text-left font-mono text-[10px] font-semibold tracking-widest text-slate-500">AI ANALYSIS</th>
              <th class="px-4 py-2.5 text-right font-mono text-[10px] font-semibold tracking-widest text-slate-500">TIME</th>
            </tr>
          </thead>
          <tbody>
            <template v-for="log in filtered" :key="log.id">

              <!-- Row -->
              <tr
                class="border-b border-[#222636] cursor-pointer transition-colors hover:bg-[#191d28] border-l-2"
                :class="[SEV_STYLE[log.severity].leftBar, expanded === log.id ? 'bg-[#191d28]' : '']"
                @click="toggle(log.id!)"
              >
                <td class="px-4 py-3 align-top">
                  <span
                    class="inline-block px-2 py-0.5 rounded border font-mono text-[10px] font-semibold tracking-widest"
                    :class="SEV_STYLE[log.severity].badge"
                  >{{ log.severity }}</span>
                </td>
                <td class="px-4 py-3 align-top">
                  <span class="flex items-center gap-1.5 font-mono text-[11px] text-slate-300">
                    <span class="w-1.5 h-1.5 rounded-full bg-emerald-400 shrink-0" />
                    {{ log.server || 'unknown' }}
                  </span>
                </td>
                <td class="px-4 py-3 align-top">
                  <p class="font-mono text-[11px] leading-relaxed break-words line-clamp-2"
                     :class="log.severity === 'CRITICAL' ? 'text-red-300' : 'text-amber-200/80'">
                    {{ log.message }}
                  </p>
                </td>
                <td class="px-4 py-3 align-top">
                  <p class="text-xs leading-relaxed break-words line-clamp-2 text-slate-400">
                    {{ log.analysis }}
                  </p>
                </td>
                <td class="px-4 py-3 align-top text-right">
                  <span class="block font-mono text-[11px] text-slate-200">{{ fmtRel(log.time) }}</span>
                  <span class="block font-mono text-[9px] text-slate-600 mt-0.5">{{ fmtTime(log.time) }}</span>
                </td>
              </tr>

              <!-- Expanded row -->
              <tr v-if="expanded === log.id" class="border-b border-[#222636]">
                <td colspan="5" class="p-0 bg-[#0f1219] border-l-2" :class="SEV_STYLE[log.severity].leftBar">
                  <div class="grid gap-4 p-5">
                    <div>
                      <p class="font-mono text-[10px] font-semibold tracking-widest text-slate-500 uppercase mb-2">⚠ Full Error</p>
                      <pre class="font-mono text-[12px] leading-relaxed whitespace-pre-wrap break-words p-4 rounded border border-[#222636] bg-orange-500/[0.03] text-amber-200/80">{{ log.message }}</pre>
                    </div>
                    <div>
                      <p class="font-mono text-[10px] font-semibold tracking-widest text-slate-500 uppercase mb-2">🤖 AI Analysis &amp; Suggested Fix</p>
                      <pre class="font-mono text-[12px] leading-relaxed whitespace-pre-wrap break-words p-4 rounded border border-violet-500/20 bg-violet-500/[0.04] text-slate-300">{{ log.analysis }}</pre>
                    </div>
                    <div class="flex flex-wrap gap-6 font-mono text-[11px] text-slate-500">
                      <span>Server: <b class="text-slate-300">{{ log.server }}</b></span>
                      <span>Severity: <b class="text-slate-300">{{ log.severity }}</b></span>
                      <span>Timestamp: <b class="text-slate-300">{{ fmtTime(log.time) }}</b></span>
                    </div>
                  </div>
                </td>
              </tr>

            </template>
          </tbody>
        </table>
      </div>
    </div>

  </div>
</template>