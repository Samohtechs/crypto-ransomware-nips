<template>
  <section>
    <!-- Header with Refresh Button -->
    <div class="mb-6 flex items-center justify-between">
      <div>
        <p class="text-slate-400">
          ML Model Status > Active detection models performance and deployment status
        </p>
      </div>

      <button
        @click="loadModelStatus"
        :disabled="loading"
        class="bg-slate-800 hover:bg-slate-700 transition px-5 py-3 rounded-xl font-medium disabled:opacity-50"
      >
        {{ loading ? "Refreshing..." : "Refresh" }}
      </button>
    </div>

    <!-- Loading State -->
    <div v-if="loading && !modelLoaded" class="text-slate-400">
      Loading model status...
    </div>

    <!-- Error State -->
    <div
      v-else-if="error"
      class="bg-red-500/10 border border-red-500/30 text-red-400 p-4 rounded-xl"
    >
      {{ error }}
    </div>

    <!-- Empty State -->
    <div
      v-else-if="models.length === 0"
      class="bg-slate-900 border border-slate-800 rounded-2xl p-6 text-slate-400"
    >
      No active models found.
    </div>

    <!-- Data Display -->
    <div v-else>
      <!-- Summary -->
      <div class="bg-slate-900 border border-slate-800 rounded-2xl p-5 mb-6">
        <div class="flex flex-col md:flex-row md:items-center md:justify-between gap-3">
          <div>
            <p class="text-slate-400 text-sm">Active Models</p>
            <h3 class="text-3xl font-bold mt-1 text-green-400">
              {{ activeModelCount }}
            </h3>
          </div>

          <div class="text-slate-400">
            Current mode:
            <span class="text-yellow-400 font-medium">
              {{ predictionMode }}
            </span>
          </div>
        </div>
      </div>

      <!-- Models List -->
      <div class="space-y-6">
        <div
          v-for="model in models"
          :key="`${model.model_name}-${model.version}`"
          class="bg-slate-950 border border-slate-800 rounded-2xl p-6"
        >
          <!-- Model Title -->
          <div class="flex flex-col md:flex-row md:items-center md:justify-between gap-3 mb-5">
            <div>
              <h3 class="text-xl font-bold">
                {{ model.model_name }}
                <span class="text-slate-400 text-sm font-normal">
                  {{ model.version }}
                </span>
              </h3>

              <p class="text-slate-400 text-sm mt-1">
                Dataset: {{ model.dataset || "—" }}
              </p>
            </div>

            <span
              class="px-4 py-2 rounded-xl text-sm font-medium"
              :class="model.active ? 'bg-green-500/10 text-green-400 border border-green-500/30' : 'bg-red-500/10 text-red-400 border border-red-500/30'"
            >
              {{ model.active ? "Active" : "Inactive" }}
            </span>
          </div>

          <!-- Performance Metrics Cards -->
          <div class="grid grid-cols-1 md:grid-cols-4 gap-5 mb-6">
            <div class="bg-slate-900 border border-slate-800 rounded-2xl p-5">
              <p class="text-slate-400 text-sm">Accuracy</p>
              <h3 class="text-3xl font-bold mt-2 text-green-400">
                {{ formatMetric(model.performance?.accuracy) }}%
              </h3>
            </div>

            <div class="bg-slate-900 border border-slate-800 rounded-2xl p-5">
              <p class="text-slate-400 text-sm">Precision</p>
              <h3 class="text-3xl font-bold mt-2">
                {{ formatMetric(model.performance?.precision) }}%
              </h3>
            </div>

            <div class="bg-slate-900 border border-slate-800 rounded-2xl p-5">
              <p class="text-slate-400 text-sm">Recall</p>
              <h3 class="text-3xl font-bold mt-2">
                {{ formatMetric(model.performance?.recall) }}%
              </h3>
            </div>

            <div class="bg-slate-900 border border-slate-800 rounded-2xl p-5">
              <p class="text-slate-400 text-sm">F1 Score</p>
              <h3 class="text-3xl font-bold mt-2">
                {{ formatMetric(model.performance?.f1_score) }}%
              </h3>
            </div>
          </div>

          <!-- Model Information -->
          <div class="bg-slate-900 border border-slate-800 rounded-2xl p-6">
            <h3 class="text-lg font-bold mb-5">Model Information</h3>

            <div class="grid grid-cols-1 md:grid-cols-2 gap-5">
              <InfoRow label="Model Name" :value="model.model_name" />
              <InfoRow label="Version" :value="model.version" />
              <InfoRow label="Dataset" :value="model.dataset" />
              <InfoRow label="Status" :value="model.status" />
              <InfoRow label="Last Trained" :value="formatTimestamp(model.last_trained)" />
              <InfoRow label="Prediction Mode" :value="model.prediction_mode" />
            </div>
          </div>
        </div>
      </div>
    </div>
  </section>
</template>

<script setup>
import { onMounted, onUnmounted, ref, h, defineComponent, computed } from 'vue';
import { useRouter } from 'vue-router';
import api from '../services/api';

// --- State ---
const loading = ref(false);
const error = ref('');
const modelLoaded = ref(false);
const models = ref([]);
let refreshInterval = null;

// Router for redirect
const router = useRouter();

// --- Computed ---
const activeModelCount = computed(() => models.value.length);

const predictionMode = computed(() => {
  if (!models.value.length) return 'Unknown';

  const mode = models.value[0]?.prediction_mode;

  return mode || 'Unknown';
});

// --- Helper: Format Metric ---
const formatMetric = (value) => {
  if (value === null || value === undefined || Number.isNaN(Number(value))) {
    return '0.00';
  }

  return Number(value).toFixed(2);
};

// --- Helper: Format Timestamp ---
const formatTimestamp = (isoString) => {
  if (!isoString) return '—';

  try {
    return new Intl.DateTimeFormat(undefined, {
      year: 'numeric',
      month: 'short',
      day: 'numeric',
      hour: '2-digit',
      minute: '2-digit',
      second: '2-digit',
    }).format(new Date(isoString));
  } catch {
    return isoString;
  }
};

// --- Auth Handling ---
const clearAuthAndRedirect = (reason) => {
  sessionStorage.setItem('authWarning', reason);
  localStorage.removeItem('access_token');
  sessionStorage.removeItem('access_token');

  error.value = `${reason} Redirecting to login...`;

  setTimeout(() => {
    router.push('/login');
  }, 1500);
};

// --- InfoRow Component ---
const InfoRow = defineComponent({
  props: {
    label: String,
    value: [String, Number],
  },
  setup(props) {
    return () =>
      h('div', { class: 'flex justify-between border-b border-slate-800 pb-3 gap-4' }, [
        h('span', { class: 'text-slate-400' }, props.label),
        h('span', { class: 'text-white text-right' }, props.value ?? '—'),
      ]);
  },
});

// --- Normalize Backend Response ---
const normalizeModelResponse = (data) => {
  // New response format:
  // { active_model_count: 2, models: [...] }
  if (Array.isArray(data.models)) {
    return data.models;
  }

  // Backward compatibility for old single-model response
  if (data.model_name) {
    return [data];
  }

  return [];
};

// --- Fetch Model Status ---
const loadModelStatus = async () => {
  if (loading.value) return;

  loading.value = true;
  error.value = '';

  try {
    const response = await api.get('/api/v1/ml_model/info');
    const data = response.data;

    models.value = normalizeModelResponse(data).map((model) => ({
      model_name: model.model_name || '—',
      version: model.version || '—',
      dataset: model.dataset || '—',
      performance: {
        accuracy: model.performance?.accuracy ?? 0,
        precision: model.performance?.precision ?? 0,
        recall: model.performance?.recall ?? 0,
        f1_score: model.performance?.f1_score ?? 0,
      },
      last_trained: model.last_trained || null,
      status: model.status || 'Unknown',
      active: model.active ?? false,
      prediction_mode: model.prediction_mode || 'Unknown',
    }));

    modelLoaded.value = true;
  } catch (err) {
    console.error('Model status error:', err);

    if (err.response?.status === 401) {
      clearAuthAndRedirect('Your session has expired. Please log in again.');
    } else if (err.response?.status === 403) {
      clearAuthAndRedirect('Access forbidden. Your token may be invalid or you lack permissions.');
    } else if (err.code === 'ERR_NETWORK') {
      error.value = 'Cannot connect to backend. Is the server running?';
    } else if (err.response?.status === 404) {
      error.value = 'No active model found in the database.';
      models.value = [];
    } else {
      error.value = 'Failed to fetch model status. Please try again later.';
    }
  } finally {
    loading.value = false;
  }
};

// --- Polling ---
const startPolling = () => {
  if (refreshInterval) clearInterval(refreshInterval);

  refreshInterval = setInterval(() => {
    if (!loading.value && !error.value?.includes('Redirecting')) {
      loadModelStatus();
    }
  }, 60000);
};

// --- Lifecycle ---
onMounted(() => {
  loadModelStatus();
  startPolling();
});

onUnmounted(() => {
  if (refreshInterval) clearInterval(refreshInterval);
});
</script>