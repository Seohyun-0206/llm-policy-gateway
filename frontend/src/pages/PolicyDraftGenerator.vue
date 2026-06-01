<script setup lang="ts">
import { computed, onMounted, ref } from 'vue'
import { CheckCircleIcon, ChevronDownIcon, ChevronUpIcon, SaveIcon, SparklesIcon, XCircleIcon } from 'lucide-vue-next'
import AppSelect, { SelectOption } from '../components/common/AppSelect.vue'
import { LLMModel, PolicyDraft, ServiceFeature, TierRecommendation, useApi } from '../composables/useApi'

const api = useApi()
const models = ref<LLMModel[]>([])
const features = ref<ServiceFeature[]>([])
const recommendations = ref<TierRecommendation[]>([])
const loading = ref(false)
const generating = ref(false)
const saving = ref(false)
const error = ref('')
const saveSuccess = ref('')
const draft = ref<PolicyDraft | null>(null)
const expandedSections = ref<Set<string>>(new Set(['routing-rules']))

// Form
const draftName = ref('My Routing Policy')
const preset = ref('balanced')
const selectedModelIds = ref<number[]>([])
const selectedFeatureIds = ref<number[]>([])
const tierOverrides = ref<Record<string, string>>({})

const PRESET_OPTIONS: SelectOption[] = [
  { value: 'balanced', label: 'Balanced' },
  { value: 'cost-first', label: 'Cost First' },
  { value: 'quality-first', label: 'Quality First' },
  { value: 'privacy-first', label: 'Privacy First' },
]
const TIER_OPTIONS: SelectOption[] = [
  { value: 'lightweight', label: 'Lightweight' },
  { value: 'standard', label: 'Standard' },
  { value: 'advanced', label: 'Advanced' },
  { value: 'long_context', label: 'Long Context' },
  { value: 'structured', label: 'Structured' },
]
const TIER_COLORS: Record<string, string> = {
  lightweight: 'border-emerald-500/20 bg-emerald-500/10 text-emerald-400',
  standard: 'border-sky-500/20 bg-sky-500/10 text-sky-300',
  advanced: 'border-violet-500/20 bg-violet-500/10 text-violet-400',
  long_context: 'border-amber-500/20 bg-amber-500/10 text-amber-400',
  structured: 'border-rose-500/20 bg-rose-500/10 text-rose-400',
}

const suggestedTierMap = computed(() =>
  Object.fromEntries(recommendations.value.map((r) => [r.model_id, r.suggested_tier]))
)

const selectedModels = computed(() => models.value.filter((m) => selectedModelIds.value.includes(m.id)))

function getTierForModel(modelId: number): string {
  return tierOverrides.value[String(modelId)] ?? suggestedTierMap.value[modelId] ?? 'standard'
}

function toggleModel(id: number) {
  const idx = selectedModelIds.value.indexOf(id)
  if (idx >= 0) {
    selectedModelIds.value.splice(idx, 1)
    delete tierOverrides.value[String(id)]
  } else {
    selectedModelIds.value.push(id)
  }
}

function toggleFeature(id: number) {
  const idx = selectedFeatureIds.value.indexOf(id)
  if (idx >= 0) selectedFeatureIds.value.splice(idx, 1)
  else selectedFeatureIds.value.push(id)
}

function toggleSection(key: string) {
  if (expandedSections.value.has(key)) expandedSections.value.delete(key)
  else expandedSections.value.add(key)
}

const canGenerate = computed(
  () => draftName.value.trim() && selectedModelIds.value.length > 0 && selectedFeatureIds.value.length > 0
)

async function generate() {
  generating.value = true
  error.value = ''
  draft.value = null
  saveSuccess.value = ''
  try {
    draft.value = await api.generatePolicyDraft({
      name: draftName.value.trim(),
      preset: preset.value,
      model_ids: selectedModelIds.value,
      feature_ids: selectedFeatureIds.value,
      tier_overrides: tierOverrides.value,
    })
    expandedSections.value = new Set(['routing-rules'])
  } catch (err) {
    error.value = err instanceof Error ? err.message : 'Failed to generate draft'
  } finally {
    generating.value = false
  }
}

async function saveAsPolicy() {
  if (!draft.value) return
  saving.value = true
  error.value = ''
  saveSuccess.value = ''
  try {
    const result = await api.savePolicyDraft(draft.value.id)
    saveSuccess.value = `Policy "${result.policy_name}" saved successfully (ID: ${result.policy_id})`
    draft.value = { ...draft.value, is_saved: true }
  } catch (err) {
    error.value = err instanceof Error ? err.message : 'Failed to save policy'
  } finally {
    saving.value = false
  }
}

function getDraftModelName(modelId: number): string {
  return models.value.find((m) => m.id === modelId)?.display_name ?? `Model #${modelId}`
}

function getDraftFeatureName(featureId: number): string {
  return features.value.find((f) => f.id === Number(featureId))?.name ?? `Feature #${featureId}`
}

async function load() {
  loading.value = true
  try {
    const [m, f, r] = await Promise.all([api.getModels(), api.getServiceFeatures(), api.getTierRecommendations()])
    models.value = m.filter((m) => m.is_active)
    features.value = f.filter((f) => f.is_active)
    recommendations.value = r
    selectedFeatureIds.value = f.filter((f) => f.is_active).map((f) => f.id)
  } finally {
    loading.value = false
  }
}

onMounted(load)
</script>

<template>
  <div class="p-6 lg:p-8">
    <div class="mb-6">
      <p class="mb-1 text-xs font-semibold uppercase tracking-widest text-indigo-400">Policy Draft — Step 4</p>
      <h2 class="text-2xl font-bold text-zinc-100">Policy Draft Generator</h2>
      <p class="mt-1 text-sm text-zinc-500">모델과 서비스 기능을 선택하면 라우팅 정책 초안을 자동으로 생성합니다.</p>
    </div>

    <div v-if="loading" class="flex items-center justify-center py-20 text-zinc-500">
      <div class="h-5 w-5 animate-spin rounded-full border-2 border-zinc-700 border-t-indigo-500 mr-3"></div>
      Loading...
    </div>

    <div v-else class="grid gap-6 xl:grid-cols-[400px_minmax(0,1fr)]">
      <!-- Config Panel -->
      <div class="space-y-5">
        <div class="rounded-xl border border-zinc-800 bg-zinc-900 p-5">
          <h3 class="mb-4 text-sm font-semibold text-zinc-200">1. Draft Configuration</h3>
          <div class="space-y-4">
            <label class="block">
              <span class="mb-1.5 block text-xs font-medium text-zinc-400">Policy Name</span>
              <input v-model="draftName" class="w-full rounded-lg border border-zinc-700 bg-zinc-800 px-3 py-2.5 text-sm text-zinc-200 outline-none focus:border-indigo-500 focus:ring-1 focus:ring-indigo-500/50" />
            </label>
            <label class="block">
              <span class="mb-1.5 block text-xs font-medium text-zinc-400">Preset</span>
              <AppSelect v-model="preset" :options="PRESET_OPTIONS" />
            </label>
          </div>
        </div>

        <div class="rounded-xl border border-zinc-800 bg-zinc-900 p-5">
          <h3 class="mb-4 text-sm font-semibold text-zinc-200">2. Select Models</h3>
          <div v-if="models.length === 0" class="text-sm text-zinc-600">등록된 활성 모델이 없습니다.</div>
          <div v-else class="space-y-2">
            <div v-for="model in models" :key="model.id" class="rounded-lg border transition-colors"
              :class="selectedModelIds.includes(model.id) ? 'border-indigo-500/40 bg-indigo-500/5' : 'border-zinc-800 bg-zinc-800/40 hover:bg-zinc-800/60'">
              <label class="flex cursor-pointer items-start gap-3 p-3">
                <input type="checkbox" class="mt-0.5 h-4 w-4 rounded border-zinc-600 bg-zinc-800 text-indigo-600" :checked="selectedModelIds.includes(model.id)" @change="toggleModel(model.id)" />
                <div class="flex-1 min-w-0">
                  <p class="truncate text-sm font-medium text-zinc-200">{{ model.display_name }}</p>
                  <p class="text-xs text-zinc-500">{{ model.provider }} · Q{{ model.quality_level }} S{{ model.speed_level }} C{{ model.cost_level }}</p>
                </div>
                <span :class="['shrink-0 rounded border px-1.5 py-0.5 text-xs', TIER_COLORS[getTierForModel(model.id)] ?? 'border-zinc-700 text-zinc-400']">
                  {{ getTierForModel(model.id) }}
                </span>
              </label>
              <!-- Tier override -->
              <div v-if="selectedModelIds.includes(model.id)" class="border-t border-zinc-800 px-3 pb-3 pt-2">
                <p class="mb-1.5 text-xs text-zinc-500">Override Tier</p>
                <AppSelect :model-value="getTierForModel(model.id)" :options="TIER_OPTIONS" @update:model-value="(v) => { tierOverrides[String(model.id)] = v }" />
              </div>
            </div>
          </div>
        </div>

        <div class="rounded-xl border border-zinc-800 bg-zinc-900 p-5">
          <h3 class="mb-4 text-sm font-semibold text-zinc-200">3. Select Service Features</h3>
          <div v-if="features.length === 0" class="text-sm text-zinc-600">등록된 서비스 기능이 없습니다.</div>
          <div v-else class="space-y-2">
            <label v-for="feature in features" :key="feature.id" class="flex cursor-pointer items-start gap-3 rounded-lg border border-zinc-800 bg-zinc-800/40 p-3 hover:bg-zinc-800/60 transition-colors"
              :class="selectedFeatureIds.includes(feature.id) ? '!border-indigo-500/40 !bg-indigo-500/5' : ''">
              <input type="checkbox" class="mt-0.5 h-4 w-4 rounded border-zinc-600 bg-zinc-800 text-indigo-600" :checked="selectedFeatureIds.includes(feature.id)" @change="toggleFeature(feature.id)" />
              <div class="flex-1 min-w-0">
                <p class="text-sm font-medium text-zinc-200">{{ feature.name }}</p>
                <div class="mt-1 flex flex-wrap items-center gap-1.5">
                  <span :class="['rounded border px-1.5 py-0.5 text-xs', TIER_COLORS[feature.required_tier] ?? 'border-zinc-700 text-zinc-400']">{{ feature.required_tier }}</span>
                  <span class="text-xs text-zinc-500">{{ feature.condition_key }}</span>
                </div>
              </div>
            </label>
          </div>
        </div>

        <button
          class="flex w-full items-center justify-center gap-2 rounded-xl bg-indigo-600 px-4 py-3 text-sm font-semibold text-white shadow-lg shadow-indigo-500/20 transition hover:bg-indigo-500 disabled:cursor-not-allowed disabled:opacity-50"
          :disabled="!canGenerate || generating"
          type="button"
          @click="generate"
        >
          <div v-if="generating" class="h-4 w-4 animate-spin rounded-full border-2 border-white/30 border-t-white"></div>
          <SparklesIcon v-else class="h-4 w-4" />
          {{ generating ? 'Generating...' : 'Generate Draft' }}
        </button>
      </div>

      <!-- Draft Panel -->
      <div>
        <div v-if="error" class="mb-4 rounded-lg border border-red-500/20 bg-red-500/10 px-3 py-2.5 text-sm text-red-400">{{ error }}</div>
        <div v-if="saveSuccess" class="mb-4 flex items-center gap-2 rounded-lg border border-emerald-500/20 bg-emerald-500/10 px-3 py-2.5 text-sm text-emerald-400">
          <CheckCircleIcon class="h-4 w-4 shrink-0" />{{ saveSuccess }}
        </div>

        <div v-if="!draft" class="flex flex-col items-center justify-center rounded-xl border border-zinc-800 bg-zinc-900/50 py-24 text-center">
          <SparklesIcon class="mb-4 h-12 w-12 text-zinc-700" />
          <p class="text-sm font-medium text-zinc-400">Configure and generate a draft</p>
          <p class="mt-1 text-sm text-zinc-600">모델과 기능을 선택한 후 Generate를 클릭하세요.</p>
        </div>

        <div v-else class="space-y-4">
          <!-- Header -->
          <div class="flex items-start justify-between gap-4 rounded-xl border border-zinc-800 bg-zinc-900 p-5">
            <div>
              <div class="flex items-center gap-2">
                <h3 class="font-semibold text-zinc-100">{{ draft.name }}</h3>
                <span class="rounded-md border border-indigo-500/20 bg-indigo-500/10 px-2 py-0.5 text-xs text-indigo-400">{{ draft.preset }}</span>
                <span v-if="draft.is_saved" class="rounded-md border border-emerald-500/20 bg-emerald-500/10 px-2 py-0.5 text-xs text-emerald-400">saved</span>
              </div>
              <p class="mt-1 text-xs text-zinc-500">Draft #{{ draft.id }} · {{ draft.routing_rules.length }} routing rules · {{ draft.threshold_rules.length }} threshold rules</p>
            </div>
            <button v-if="!draft.is_saved"
              class="flex shrink-0 items-center gap-2 rounded-lg bg-emerald-600 px-4 py-2 text-sm font-semibold text-white shadow-lg shadow-emerald-500/20 transition hover:bg-emerald-500 disabled:opacity-50"
              :disabled="saving"
              type="button"
              @click="saveAsPolicy"
            >
              <div v-if="saving" class="h-4 w-4 animate-spin rounded-full border-2 border-white/30 border-t-white"></div>
              <SaveIcon v-else class="h-4 w-4" />
              {{ saving ? 'Saving...' : 'Save as Policy' }}
            </button>
          </div>

          <!-- Missing Coverage -->
          <div v-if="draft.missing_coverage.length" class="rounded-xl border border-amber-500/20 bg-amber-500/5 p-4">
            <p class="mb-2 text-xs font-semibold uppercase tracking-wider text-amber-400">⚠ Missing Coverage</p>
            <ul class="space-y-1">
              <li v-for="mc in draft.missing_coverage" :key="mc.feature_id" class="text-sm text-amber-300">{{ mc.message }}</li>
            </ul>
          </div>

          <!-- Summary -->
          <div class="rounded-xl border border-zinc-800 bg-zinc-900 p-5">
            <p class="mb-3 text-xs font-semibold uppercase tracking-wider text-zinc-500">Policy Summary</p>
            <pre class="whitespace-pre-wrap rounded-lg bg-zinc-800/60 p-3 text-xs leading-relaxed text-zinc-300">{{ draft.summary_text }}</pre>
          </div>

          <!-- Tier Assignments -->
          <div class="rounded-xl border border-zinc-800 bg-zinc-900">
            <button class="flex w-full items-center justify-between px-5 py-4 text-left" type="button" @click="toggleSection('tiers')">
              <p class="text-sm font-semibold text-zinc-200">Tier Assignments <span class="ml-1.5 rounded-full bg-zinc-800 px-2 py-0.5 text-xs text-zinc-400">{{ Object.keys(draft.tier_assignments).length }}</span></p>
              <ChevronDownIcon v-if="!expandedSections.has('tiers')" class="h-4 w-4 text-zinc-500" />
              <ChevronUpIcon v-else class="h-4 w-4 text-zinc-500" />
            </button>
            <div v-if="expandedSections.has('tiers')" class="border-t border-zinc-800 px-5 pb-4 pt-3">
              <div class="grid gap-2 sm:grid-cols-2">
                <div v-for="(tier, modelId) in draft.tier_assignments" :key="modelId" class="flex items-center justify-between rounded-lg bg-zinc-800/40 px-3 py-2">
                  <p class="truncate text-sm text-zinc-300">{{ getDraftModelName(Number(modelId)) }}</p>
                  <span :class="['ml-2 shrink-0 rounded border px-1.5 py-0.5 text-xs font-medium', TIER_COLORS[tier] ?? 'border-zinc-700 text-zinc-400']">{{ tier }}</span>
                </div>
              </div>
            </div>
          </div>

          <!-- Routing Rules -->
          <div class="rounded-xl border border-zinc-800 bg-zinc-900">
            <button class="flex w-full items-center justify-between px-5 py-4 text-left" type="button" @click="toggleSection('routing-rules')">
              <p class="text-sm font-semibold text-zinc-200">Routing Rules <span class="ml-1.5 rounded-full bg-zinc-800 px-2 py-0.5 text-xs text-zinc-400">{{ draft.routing_rules.length }}</span></p>
              <ChevronDownIcon v-if="!expandedSections.has('routing-rules')" class="h-4 w-4 text-zinc-500" />
              <ChevronUpIcon v-else class="h-4 w-4 text-zinc-500" />
            </button>
            <div v-if="expandedSections.has('routing-rules')" class="border-t border-zinc-800 px-5 pb-4 pt-3 space-y-2">
              <div v-for="rule in draft.routing_rules" :key="rule.rule_id" class="rounded-lg border border-zinc-800 bg-zinc-800/40 px-3 py-2.5">
                <div class="flex items-center gap-2">
                  <span class="text-xs font-mono text-zinc-500">{{ rule.rule_id }}</span>
                  <span class="text-sm font-medium text-zinc-200">{{ rule.name }}</span>
                </div>
                <div class="mt-1.5 flex flex-wrap items-center gap-2 text-xs">
                  <span class="text-zinc-500">condition: <span class="text-zinc-300">{{ rule.condition_key }}</span></span>
                  <span class="text-zinc-500">→</span>
                  <span :class="['rounded border px-1.5 py-0.5', TIER_COLORS[rule.target_tier] ?? 'border-zinc-700 text-zinc-400']">{{ rule.target_tier }}</span>
                  <span class="text-zinc-600">priority {{ rule.priority }}</span>
                </div>
              </div>
            </div>
          </div>

          <!-- Threshold Rules -->
          <div class="rounded-xl border border-zinc-800 bg-zinc-900">
            <button class="flex w-full items-center justify-between px-5 py-4 text-left" type="button" @click="toggleSection('threshold-rules')">
              <p class="text-sm font-semibold text-zinc-200">Threshold Rules <span class="ml-1.5 rounded-full bg-zinc-800 px-2 py-0.5 text-xs text-zinc-400">{{ draft.threshold_rules.length }}</span></p>
              <ChevronDownIcon v-if="!expandedSections.has('threshold-rules')" class="h-4 w-4 text-zinc-500" />
              <ChevronUpIcon v-else class="h-4 w-4 text-zinc-500" />
            </button>
            <div v-if="expandedSections.has('threshold-rules')" class="border-t border-zinc-800 px-5 pb-4 pt-3 space-y-2">
              <div v-for="rule in draft.threshold_rules" :key="rule.rule_id" class="rounded-lg border border-zinc-800 bg-zinc-800/40 px-3 py-2.5">
                <div class="flex items-center gap-2">
                  <span class="text-xs font-mono text-zinc-500">{{ rule.rule_id }}</span>
                  <span class="text-sm font-medium text-zinc-200">{{ rule.name }}</span>
                </div>
                <p class="mt-1 text-xs text-zinc-500">{{ rule.metric_key }} {{ rule.operator }} {{ rule.threshold_value }} → {{ rule.action_on_trigger }} <span v-if="rule.target_tier">({{ rule.target_tier }})</span></p>
              </div>
            </div>
          </div>

          <!-- Validation + Recovery -->
          <div v-if="draft.validation_rules.length || draft.recovery_strategies.length" class="rounded-xl border border-zinc-800 bg-zinc-900">
            <button class="flex w-full items-center justify-between px-5 py-4 text-left" type="button" @click="toggleSection('validation')">
              <p class="text-sm font-semibold text-zinc-200">Validation & Recovery <span class="ml-1.5 rounded-full bg-zinc-800 px-2 py-0.5 text-xs text-zinc-400">{{ draft.validation_rules.length + draft.recovery_strategies.length }}</span></p>
              <ChevronDownIcon v-if="!expandedSections.has('validation')" class="h-4 w-4 text-zinc-500" />
              <ChevronUpIcon v-else class="h-4 w-4 text-zinc-500" />
            </button>
            <div v-if="expandedSections.has('validation')" class="border-t border-zinc-800 px-5 pb-4 pt-3 space-y-2">
              <div v-for="rule in draft.validation_rules" :key="rule.rule_id" class="rounded-lg border border-zinc-800 bg-zinc-800/40 px-3 py-2.5">
                <div class="flex items-center gap-2">
                  <span class="text-xs font-mono text-zinc-500">{{ rule.rule_id }}</span>
                  <span class="text-sm font-medium text-zinc-200">{{ rule.name }}</span>
                  <span class="rounded border border-sky-500/20 bg-sky-500/10 px-1.5 py-0.5 text-xs text-sky-300">validation</span>
                </div>
                <p class="mt-1 text-xs text-zinc-500">{{ rule.validation_type }} · on fail: {{ rule.action_on_fail }} · max retries: {{ rule.max_retries }}</p>
              </div>
              <div v-for="strat in draft.recovery_strategies" :key="strat.strategy_id" class="rounded-lg border border-zinc-800 bg-zinc-800/40 px-3 py-2.5">
                <div class="flex items-center gap-2">
                  <span class="text-xs font-mono text-zinc-500">{{ strat.strategy_id }}</span>
                  <span class="text-sm font-medium text-zinc-200">{{ strat.name }}</span>
                  <span class="rounded border border-rose-500/20 bg-rose-500/10 px-1.5 py-0.5 text-xs text-rose-300">recovery</span>
                </div>
                <p class="mt-1 text-xs text-zinc-500">trigger: {{ strat.trigger_event }} · action: {{ strat.action }} · max retries: {{ strat.max_retries }}</p>
              </div>
            </div>
          </div>

          <!-- Feature Model Map -->
          <div class="rounded-xl border border-zinc-800 bg-zinc-900">
            <button class="flex w-full items-center justify-between px-5 py-4 text-left" type="button" @click="toggleSection('feature-map')">
              <p class="text-sm font-semibold text-zinc-200">Feature → Model Assignment</p>
              <ChevronDownIcon v-if="!expandedSections.has('feature-map')" class="h-4 w-4 text-zinc-500" />
              <ChevronUpIcon v-else class="h-4 w-4 text-zinc-500" />
            </button>
            <div v-if="expandedSections.has('feature-map')" class="border-t border-zinc-800 px-5 pb-4 pt-3 space-y-3">
              <div v-for="(modelIds, featureId) in draft.feature_model_map" :key="featureId">
                <p class="mb-1.5 text-xs font-medium text-zinc-400">{{ getDraftFeatureName(Number(featureId)) }}</p>
                <div class="space-y-1">
                  <div v-for="(modelId, idx) in (modelIds as number[])" :key="modelId" class="flex items-center gap-2 rounded-md bg-zinc-800/40 px-2.5 py-1.5">
                    <span class="text-xs font-bold" :class="idx === 0 ? 'text-indigo-400' : 'text-zinc-600'">#{{ idx + 1 }}</span>
                    <span class="text-sm text-zinc-300">{{ getDraftModelName(modelId) }}</span>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
