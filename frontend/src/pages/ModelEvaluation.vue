<script setup lang="ts">
import { onMounted, ref } from 'vue'
import { BeakerIcon, CpuIcon } from 'lucide-vue-next'
import AdminDataTable from '../components/common/AdminDataTable.vue'
import { LLMModel, useApi } from '../composables/useApi'

const api = useApi()
const models = ref<LLMModel[]>([])
const loading = ref(false)

async function loadModels() {
  loading.value = true
  try {
    models.value = (await api.getModels()).filter((m) => m.is_active)
  } finally {
    loading.value = false
  }
}

onMounted(loadModels)
</script>

<template>
  <div class="p-6 lg:p-8">
    <div class="mb-6 flex items-start justify-between gap-4">
      <div>
        <p class="mb-1 text-xs font-semibold uppercase tracking-widest text-indigo-400">Policy Draft — Step 0</p>
        <h2 class="text-2xl font-bold text-zinc-100">Model Evaluation</h2>
        <p class="mt-1 text-sm text-zinc-500">등록된 모델 목록을 확인합니다. 실제 평가 실행은 2단계에서 추가될 예정입니다.</p>
      </div>
    </div>

    <AdminDataTable :loading="loading" :is-empty="models.length === 0">
      <template #head>
        <th class="px-5 py-3.5 text-left text-xs font-medium uppercase tracking-wider text-zinc-500">Model</th>
        <th class="px-5 py-3.5 text-left text-xs font-medium uppercase tracking-wider text-zinc-500">Provider</th>
        <th class="px-5 py-3.5 text-left text-xs font-medium uppercase tracking-wider text-zinc-500">Role</th>
        <th class="px-5 py-3.5 text-left text-xs font-medium uppercase tracking-wider text-zinc-500">Current Tier</th>
        <th class="px-5 py-3.5 text-left text-xs font-medium uppercase tracking-wider text-zinc-500">Quality</th>
        <th class="px-5 py-3.5 text-left text-xs font-medium uppercase tracking-wider text-zinc-500">Speed</th>
        <th class="px-5 py-3.5 text-left text-xs font-medium uppercase tracking-wider text-zinc-500">Cost</th>
        <th class="px-5 py-3.5 text-left text-xs font-medium uppercase tracking-wider text-zinc-500">Context</th>
        <th class="px-5 py-3.5 text-left text-xs font-medium uppercase tracking-wider text-zinc-500">Privacy</th>
      </template>

      <template #empty>
        <BeakerIcon class="mx-auto mb-4 h-12 w-12 text-zinc-700" />
        <h3 class="mb-1 text-sm font-semibold text-zinc-300">No active models</h3>
        <p class="text-sm text-zinc-600">Models 화면에서 모델을 등록하고 활성화하세요.</p>
      </template>

      <tr v-for="model in models" :key="model.id" class="transition-colors hover:bg-zinc-800/30">
        <td class="px-5 py-3.5">
          <p class="font-medium text-zinc-200">{{ model.display_name }}</p>
          <p class="text-xs text-zinc-500">{{ model.provider }}/{{ model.name }}</p>
        </td>
        <td class="whitespace-nowrap px-5 py-3.5 text-sm text-zinc-300 capitalize">{{ model.provider }}</td>
        <td class="whitespace-nowrap px-5 py-3.5">
          <span class="rounded-md border border-zinc-700 bg-zinc-800 px-2 py-0.5 text-xs text-zinc-400">{{ model.role }}</span>
        </td>
        <td class="whitespace-nowrap px-5 py-3.5">
          <span class="rounded-md border border-sky-500/20 bg-sky-500/10 px-2 py-0.5 text-xs font-medium text-sky-300">{{ model.model_tier }}</span>
        </td>
        <td class="whitespace-nowrap px-5 py-3.5">
          <div class="flex items-center gap-1.5">
            <div class="h-1.5 w-16 overflow-hidden rounded-full bg-zinc-800">
              <div class="h-full rounded-full bg-indigo-500" :style="{ width: `${(model.quality_level / 5) * 100}%` }" />
            </div>
            <span class="text-xs text-zinc-400">{{ model.quality_level }}/5</span>
          </div>
        </td>
        <td class="whitespace-nowrap px-5 py-3.5">
          <div class="flex items-center gap-1.5">
            <div class="h-1.5 w-16 overflow-hidden rounded-full bg-zinc-800">
              <div class="h-full rounded-full bg-emerald-500" :style="{ width: `${(model.speed_level / 5) * 100}%` }" />
            </div>
            <span class="text-xs text-zinc-400">{{ model.speed_level }}/5</span>
          </div>
        </td>
        <td class="whitespace-nowrap px-5 py-3.5">
          <div class="flex items-center gap-1.5">
            <div class="h-1.5 w-16 overflow-hidden rounded-full bg-zinc-800">
              <div class="h-full rounded-full bg-amber-500" :style="{ width: `${(model.cost_level / 5) * 100}%` }" />
            </div>
            <span class="text-xs text-zinc-400">{{ model.cost_level }}/5</span>
          </div>
        </td>
        <td class="whitespace-nowrap px-5 py-3.5 text-sm text-zinc-300">{{ model.context_window.toLocaleString() }}</td>
        <td class="whitespace-nowrap px-5 py-3.5">
          <span :class="[
            'rounded-md border px-2 py-0.5 text-xs font-medium',
            model.privacy_level === 'local'
              ? 'border-emerald-500/20 bg-emerald-500/10 text-emerald-400'
              : 'border-amber-500/20 bg-amber-500/10 text-amber-400'
          ]">{{ model.privacy_level }}</span>
        </td>
      </tr>
    </AdminDataTable>
  </div>
</template>
