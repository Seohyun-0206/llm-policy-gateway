<script setup lang="ts">
import { computed, onMounted, ref } from 'vue'
import { EditIcon, NetworkIcon, PlusIcon, SearchIcon, TrashIcon } from 'lucide-vue-next'
import AdminDataTable from '../components/common/AdminDataTable.vue'
import AppSelect, { SelectOption } from '../components/common/AppSelect.vue'
import { ServiceFeature, ServiceFeaturePayload, useApi } from '../composables/useApi'

const api = useApi()
const features = ref<ServiceFeature[]>([])
const loading = ref(false)
const searchQuery = ref('')
const error = ref('')
const showModal = ref(false)
const selected = ref<ServiceFeature | null>(null)
const saving = ref(false)

const TIER_CHOICES: SelectOption[] = [
  { value: 'lightweight', label: 'Lightweight' },
  { value: 'standard', label: 'Standard' },
  { value: 'advanced', label: 'Advanced' },
  { value: 'long_context', label: 'Long Context' },
  { value: 'structured', label: 'Structured' },
]
const PATH_CHOICES: SelectOption[] = [
  { value: 'lightweight', label: 'Lightweight Path' },
  { value: 'standard', label: 'Standard Path' },
  { value: 'advanced', label: 'Advanced Path' },
  { value: 'long_context', label: 'Long Context Path' },
  { value: 'structured', label: 'Structured Path' },
  { value: 'escalation', label: 'Escalation Path' },
  { value: 'fallback', label: 'Fallback Path' },
]
const CONDITION_CHOICES: SelectOption[] = [
  { value: 'general', label: 'General / simple query' },
  { value: 'code', label: 'Code or technical' },
  { value: 'reasoning', label: 'Reasoning' },
  { value: 'long_context', label: 'Long context' },
  { value: 'structured_output', label: 'Structured output (SQL/JSON)' },
  { value: 'sensitive', label: 'Sensitive data' },
  { value: 'always', label: 'Always' },
]

const TIER_COLORS: Record<string, string> = {
  lightweight: 'border-emerald-500/20 bg-emerald-500/10 text-emerald-400',
  standard: 'border-sky-500/20 bg-sky-500/10 text-sky-300',
  advanced: 'border-violet-500/20 bg-violet-500/10 text-violet-400',
  long_context: 'border-amber-500/20 bg-amber-500/10 text-amber-400',
  structured: 'border-rose-500/20 bg-rose-500/10 text-rose-400',
}

const form = ref<ServiceFeaturePayload>({
  name: '', description: '', required_tier: 'standard', routing_path: 'standard',
  condition_key: 'general', main_metrics: [], sort_order: 100, is_active: true,
})
const metricsInput = ref('')

const filtered = computed(() => {
  const q = searchQuery.value.trim().toLowerCase()
  return features.value.filter(
    (f) => !q || f.name.toLowerCase().includes(q) || f.required_tier.includes(q) || f.condition_key.includes(q)
  )
})

async function load() {
  loading.value = true
  try {
    features.value = await api.getServiceFeatures()
  } finally {
    loading.value = false
  }
}

function openCreate() {
  selected.value = null
  form.value = { name: '', description: '', required_tier: 'standard', routing_path: 'standard', condition_key: 'general', main_metrics: [], sort_order: 100, is_active: true }
  metricsInput.value = ''
  showModal.value = true
}

function openEdit(f: ServiceFeature) {
  selected.value = f
  form.value = { name: f.name, description: f.description, required_tier: f.required_tier, routing_path: f.routing_path, condition_key: f.condition_key, main_metrics: [...f.main_metrics], sort_order: f.sort_order, is_active: f.is_active }
  metricsInput.value = f.main_metrics.join(', ')
  showModal.value = true
}

function closeModal() { showModal.value = false; selected.value = null }

async function save() {
  saving.value = true
  error.value = ''
  try {
    form.value.main_metrics = metricsInput.value.split(',').map((s) => s.trim()).filter(Boolean)
    if (selected.value) {
      await api.updateServiceFeature(selected.value.id, form.value)
    } else {
      await api.createServiceFeature(form.value)
    }
    closeModal()
    await load()
  } catch (err) {
    error.value = err instanceof Error ? err.message : 'Failed to save'
  } finally {
    saving.value = false
  }
}

async function remove(f: ServiceFeature) {
  if (!confirm(`Delete "${f.name}"?`)) return
  try {
    await api.deleteServiceFeature(f.id)
    await load()
  } catch (err) {
    error.value = err instanceof Error ? err.message : 'Failed to delete'
  }
}

onMounted(load)
</script>

<template>
  <div class="p-6 lg:p-8">
    <div class="mb-6 flex items-center justify-between gap-4">
      <div>
        <p class="mb-1 text-xs font-semibold uppercase tracking-widest text-indigo-400">Policy Draft — Step 2</p>
        <h2 class="text-2xl font-bold text-zinc-100">Service Features</h2>
        <p class="mt-1 text-sm text-zinc-500">서비스 기능별 필요 Tier와 라우팅 Path를 정의합니다.</p>
      </div>
      <button
        class="flex items-center gap-2 rounded-lg bg-indigo-600 px-4 py-2.5 text-sm font-semibold text-white shadow-lg shadow-indigo-500/20 transition hover:bg-indigo-500"
        type="button"
        @click="openCreate"
      >
        <PlusIcon class="h-4 w-4" /> Add Feature
      </button>
    </div>

    <div class="mb-5 relative">
      <SearchIcon class="absolute left-3 top-1/2 h-4 w-4 -translate-y-1/2 text-zinc-500" />
      <input v-model="searchQuery" class="w-full max-w-sm rounded-lg border border-zinc-700 bg-zinc-800 py-2.5 pl-9 pr-4 text-sm text-zinc-200 placeholder-zinc-600 outline-none transition focus:border-indigo-500 focus:ring-1 focus:ring-indigo-500/50" placeholder="Search feature..." type="text" />
    </div>

    <div v-if="error" class="mb-4 rounded-lg border border-red-500/20 bg-red-500/10 px-3 py-2.5 text-sm text-red-400">{{ error }}</div>

    <AdminDataTable :loading="loading" :is-empty="filtered.length === 0">
      <template #head>
        <th class="px-5 py-3.5 text-left text-xs font-medium uppercase tracking-wider text-zinc-500">Feature</th>
        <th class="px-5 py-3.5 text-left text-xs font-medium uppercase tracking-wider text-zinc-500">Required Tier</th>
        <th class="px-5 py-3.5 text-left text-xs font-medium uppercase tracking-wider text-zinc-500">Routing Path</th>
        <th class="px-5 py-3.5 text-left text-xs font-medium uppercase tracking-wider text-zinc-500">Condition</th>
        <th class="px-5 py-3.5 text-left text-xs font-medium uppercase tracking-wider text-zinc-500">Main Metrics</th>
        <th class="px-5 py-3.5 text-left text-xs font-medium uppercase tracking-wider text-zinc-500">Status</th>
        <th class="px-5 py-3.5 text-left text-xs font-medium uppercase tracking-wider text-zinc-500">Actions</th>
      </template>

      <template #empty>
        <NetworkIcon class="mx-auto mb-4 h-12 w-12 text-zinc-700" />
        <h3 class="mb-1 text-sm font-semibold text-zinc-300">No service features</h3>
        <p class="mb-4 text-sm text-zinc-600">서비스 기능을 추가하세요.</p>
        <button class="rounded-lg bg-indigo-600 px-4 py-2 text-sm font-semibold text-white hover:bg-indigo-500" @click="openCreate">Add Feature</button>
      </template>

      <tr v-for="f in filtered" :key="f.id" class="transition-colors hover:bg-zinc-800/30">
        <td class="px-5 py-3.5">
          <p class="font-medium text-zinc-200">{{ f.name }}</p>
          <p class="max-w-xs truncate text-xs text-zinc-500">{{ f.description || '-' }}</p>
        </td>
        <td class="whitespace-nowrap px-5 py-3.5">
          <span :class="['rounded-md border px-2 py-0.5 text-xs font-medium', TIER_COLORS[f.required_tier] ?? 'border-zinc-700 bg-zinc-800 text-zinc-300']">{{ f.required_tier }}</span>
        </td>
        <td class="whitespace-nowrap px-5 py-3.5 text-sm text-zinc-300">{{ f.routing_path }}</td>
        <td class="whitespace-nowrap px-5 py-3.5 text-sm text-zinc-400">{{ f.condition_key }}</td>
        <td class="px-5 py-3.5">
          <div class="flex flex-wrap gap-1">
            <span v-for="metric in f.main_metrics" :key="metric" class="rounded border border-zinc-700 bg-zinc-800 px-1.5 py-0.5 text-xs text-zinc-400">{{ metric }}</span>
            <span v-if="!f.main_metrics.length" class="text-xs text-zinc-600">-</span>
          </div>
        </td>
        <td class="whitespace-nowrap px-5 py-3.5">
          <span :class="['rounded-md border px-2 py-0.5 text-xs font-medium', f.is_active ? 'border-emerald-500/20 bg-emerald-500/10 text-emerald-400' : 'border-zinc-700 bg-zinc-800 text-zinc-500']">{{ f.is_active ? 'active' : 'inactive' }}</span>
        </td>
        <td class="whitespace-nowrap px-5 py-3.5 flex gap-1">
          <button class="rounded-md p-1.5 text-zinc-500 transition-colors hover:bg-zinc-700 hover:text-zinc-200" title="Edit" type="button" @click="openEdit(f)"><EditIcon class="h-4 w-4" /></button>
          <button class="rounded-md p-1.5 text-zinc-500 transition-colors hover:bg-red-900/40 hover:text-red-400" title="Delete" type="button" @click="remove(f)"><TrashIcon class="h-4 w-4" /></button>
        </td>
      </tr>
    </AdminDataTable>

    <!-- Modal -->
    <div v-if="showModal" class="fixed inset-0 z-50 flex items-center justify-center bg-black/60 p-4">
      <div class="w-full max-w-lg rounded-xl border border-zinc-700 bg-zinc-900 shadow-2xl">
        <div class="border-b border-zinc-800 px-6 py-4">
          <h3 class="text-base font-semibold text-zinc-100">{{ selected ? 'Edit' : 'Add' }} Service Feature</h3>
        </div>
        <div class="space-y-4 p-6">
          <label class="block">
            <span class="mb-1.5 block text-xs font-medium text-zinc-400">Name</span>
            <input v-model="form.name" class="w-full rounded-lg border border-zinc-700 bg-zinc-800 px-3 py-2.5 text-sm text-zinc-200 outline-none focus:border-indigo-500 focus:ring-1 focus:ring-indigo-500/50" />
          </label>
          <label class="block">
            <span class="mb-1.5 block text-xs font-medium text-zinc-400">Description</span>
            <textarea v-model="form.description" rows="2" class="w-full rounded-lg border border-zinc-700 bg-zinc-800 px-3 py-2.5 text-sm text-zinc-200 outline-none focus:border-indigo-500 focus:ring-1 focus:ring-indigo-500/50" />
          </label>
          <div class="grid grid-cols-2 gap-4">
            <label class="block">
              <span class="mb-1.5 block text-xs font-medium text-zinc-400">Required Tier</span>
              <AppSelect v-model="form.required_tier" :options="TIER_CHOICES" />
            </label>
            <label class="block">
              <span class="mb-1.5 block text-xs font-medium text-zinc-400">Routing Path</span>
              <AppSelect v-model="form.routing_path" :options="PATH_CHOICES" />
            </label>
          </div>
          <label class="block">
            <span class="mb-1.5 block text-xs font-medium text-zinc-400">Condition Key</span>
            <AppSelect v-model="form.condition_key" :options="CONDITION_CHOICES" />
          </label>
          <label class="block">
            <span class="mb-1.5 block text-xs font-medium text-zinc-400">Main Metrics <span class="text-zinc-600">(comma-separated)</span></span>
            <input v-model="metricsInput" class="w-full rounded-lg border border-zinc-700 bg-zinc-800 px-3 py-2.5 text-sm text-zinc-200 outline-none focus:border-indigo-500 focus:ring-1 focus:ring-indigo-500/50" placeholder="Accuracy, Latency, Format Success Rate" />
          </label>
          <div class="grid grid-cols-2 gap-4">
            <label class="block">
              <span class="mb-1.5 block text-xs font-medium text-zinc-400">Sort Order</span>
              <input v-model.number="form.sort_order" type="number" class="w-full rounded-lg border border-zinc-700 bg-zinc-800 px-3 py-2.5 text-sm text-zinc-200 outline-none focus:border-indigo-500 focus:ring-1 focus:ring-indigo-500/50" />
            </label>
            <label class="flex items-center gap-3 pt-6">
              <input v-model="form.is_active" type="checkbox" class="h-4 w-4 rounded border-zinc-600 bg-zinc-800 text-indigo-600" />
              <span class="text-sm text-zinc-300">Active</span>
            </label>
          </div>
        </div>
        <div class="flex justify-end gap-3 border-t border-zinc-800 px-6 py-4">
          <button class="rounded-lg border border-zinc-700 px-4 py-2 text-sm text-zinc-300 hover:bg-zinc-800" type="button" @click="closeModal">Cancel</button>
          <button class="rounded-lg bg-indigo-600 px-4 py-2 text-sm font-semibold text-white hover:bg-indigo-500 disabled:opacity-50" :disabled="saving || !form.name" type="button" @click="save">
            {{ saving ? 'Saving...' : 'Save' }}
          </button>
        </div>
      </div>
    </div>
  </div>
</template>
