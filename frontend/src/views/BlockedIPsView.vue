<template>
  <section>
    <div class="mb-6 flex items-center justify-between">
      <div>
        <p class="text-slate-400">
          Blocked IPs > Malicious IP addresses blocked by the prevention module
        </p>
      </div>

      <div class="flex gap-3">
        <button
          @click="refreshBlockedIps"
          :disabled="loading"
          class="bg-slate-700 hover:bg-slate-600 transition px-5 py-3 rounded-xl font-medium disabled:opacity-50"
        >
          {{ loading ? "Refreshing..." : "Refresh" }}
        </button>

        <!-- <button
          @click="showForm = !showForm"
          class="bg-blue-600 hover:bg-blue-700 transition px-5 py-3 rounded-xl font-medium"
        >
          Add Manual Block
        </button> -->
      </div>
    </div>

    <!-- Add Form -->
    <div
      v-if="showForm"
      class="bg-slate-900 border border-slate-800 rounded-2xl p-6 mb-6"
    >
      <h3 class="text-lg font-bold text-white mb-4">
        Add Manual IP Block
      </h3>

      <form
        @submit.prevent="submitBlockIp"
        class="grid grid-cols-1 md:grid-cols-3 gap-4"
      >
        <input
          v-model.trim="form.ip_address"
          type="text"
          placeholder="IP Address e.g. 192.168.1.100"
          class="bg-slate-950 border border-slate-700 rounded-xl px-4 py-3 text-white outline-none focus:border-blue-500"
          required
        />

        <input
          v-model.trim="form.reason"
          type="text"
          placeholder="Reason for blocking"
          class="bg-slate-950 border border-slate-700 rounded-xl px-4 py-3 text-white outline-none focus:border-blue-500"
          required
        />

        <button
          type="submit"
          :disabled="submitting"
          class="bg-red-600 hover:bg-red-700 disabled:opacity-50 transition px-5 py-3 rounded-xl font-medium"
        >
          {{ submitting ? "Blocking..." : "Block IP" }}
        </button>
      </form>
    </div>

    <!-- Loading State -->
    <div v-if="loading && blockedIps.length === 0" class="text-slate-400">
      Loading blocked IPs...
    </div>

    <!-- Error State -->
    <div
      v-else-if="error"
      class="bg-red-500/10 border border-red-500/30 text-red-400 p-4 rounded-xl"
    >
      {{ error }}
    </div>

    <!-- Data Display -->
    <div
      v-else
      class="bg-slate-900 border border-slate-800 rounded-2xl p-6"
    >

      <!-- Search + Filters -->
        <div class="mb-5 grid grid-cols-1 gap-4 lg:grid-cols-3">
          <input
            v-model="searchQuery"
            type="text"
            placeholder="Search IP address or reason..."
            class="w-full rounded-2xl border border-white/10 bg-slate-950 px-4 py-3 text-sm text-white outline-none transition focus:border-sky-400"
          />

          <select
            v-model="modeFilter"
            class="w-full rounded-2xl border border-white/10 bg-slate-950 px-4 py-3 text-sm text-white outline-none transition focus:border-sky-400"
          >
            <option value="All">All Modes</option>
            <option value="Auto">Auto</option>
            <option value="Manual">Manual</option>
          </select>

          <select
            v-model="statusFilter"
            class="w-full rounded-2xl border border-white/10 bg-slate-950 px-4 py-3 text-sm text-white outline-none transition focus:border-sky-400"
          >
            <option value="All">All Statuses</option>
            <option value="Active">Active</option>
            <option value="Inactive">Inactive</option>
            <option value="Removed">Removed</option>
          </select>
        </div>   

      <div class="mb-5 flex items-center justify-between">
        <div>
          <h3 class="text-lg font-semibold text-white">Blocked IP Records</h3>
          <p class="mt-1 text-sm text-slate-400">
            Page {{ pagination.page }} of {{ pagination.pages }}
          </p>
        </div>

        <span class="bg-red-500/20 text-red-400 px-4 py-2 rounded-xl text-sm">
          {{ filteredBlockedIps.length }} shown / {{ pagination.total }} total
        </span>
      </div>

      <div v-if="filteredBlockedIps.length === 0" class="text-slate-400">
        No blocked IPs found.
      </div>

      <div v-else class="overflow-x-auto">
        <table class="w-full text-sm">
          <thead class="bg-slate-800 text-slate-300">
            <tr>
              <th scope="col" class="text-left p-4">IP Address (Src)</th>
              <th scope="col" class="text-left p-4">IP Address (Dst)</th>
              <th scope="col" class="text-left p-4">Reason</th>
              <th scope="col" class="text-left p-4">Blocked At</th>
              <th scope="col" class="text-left p-4">Mode</th>
              <th scope="col" class="text-left p-4">Status</th>
            </tr>
          </thead>

          <tbody>
            <tr
              v-for="ip in filteredBlockedIps"
              :key="ip.id || ip.ip_address || ip.ipAddress"
              class="border-t border-slate-800 hover:bg-slate-800/50"
            >
              <td class="p-4 font-medium text-slate-200">
                {{ ip.src_ip_address || ip.srcIpAddress || '—' }}
              </td>

              <td class="p-4 font-medium text-slate-200">
                {{ ip.dst_ip_address || ip.dstIpAddress || '—' }}
              </td>

              <td class="p-4">
                {{ ip.reason || '—' }}
              </td>

              <td class="p-4">
                {{ formatTimestamp(ip.blocked_at || ip.blockedAt) }}
              </td>

              <td class="p-4">
                <span
                  class="px-3 py-1 rounded-full text-xs font-medium"
                  :class="modeClass(ip.mode)"
                >
                  {{ ip.mode || 'Manual' }}
                </span>
              </td>

              <td class="p-4">
                <span
                  class="px-3 py-1 rounded-full text-xs font-medium"
                  :class="statusClass(ip.status)"
                >
                  {{ ip.status || 'Active' }}
                </span>
              </td>
            </tr>
          </tbody>
        </table>

        <!-- Backend Pagination Controls -->
        <div
          v-if="pagination.pages > 1"
          class="mt-5 flex flex-col gap-3 sm:flex-row sm:items-center sm:justify-between"
        >
          <p class="text-sm text-slate-400">
            Showing {{ filteredBlockedIps.length }} records on page
            {{ pagination.page }} of {{ pagination.pages }}
            · Total {{ pagination.total }}
          </p>

          <div class="flex items-center gap-2">
            <button
              @click="goToPage(pagination.page - 1)"
              :disabled="pagination.page === 1 || loading"
              class="rounded-xl border border-white/10 bg-white/[0.04] px-4 py-2 text-sm text-slate-300 transition hover:bg-white/[0.08] disabled:opacity-40"
            >
              Previous
            </button>

            <button
              v-for="page in visiblePages"
              :key="page"
              @click="goToPage(page)"
              :disabled="loading"
              :class="[
                'rounded-xl border px-4 py-2 text-sm transition disabled:opacity-40',
                page === pagination.page
                  ? 'border-sky-400/30 bg-sky-400 text-slate-950'
                  : 'border-white/10 bg-white/[0.04] text-slate-300 hover:bg-white/[0.08]'
              ]"
            >
              {{ page }}
            </button>

            <button
              @click="goToPage(pagination.page + 1)"
              :disabled="pagination.page === pagination.pages || loading"
              class="rounded-xl border border-white/10 bg-white/[0.04] px-4 py-2 text-sm text-slate-300 transition hover:bg-white/[0.08] disabled:opacity-40"
            >
              Next
            </button>
          </div>
        </div>
      </div>
    </div>
  </section>
</template>

<script setup>
import { computed, onMounted, onUnmounted, ref } from 'vue';
import { useRouter } from 'vue-router';
import api from '../services/api';

// --- State ---
const loading = ref(false);
const submitting = ref(false);
const error = ref('');
const showForm = ref(false);
const blockedIps = ref([]);

const searchQuery = ref('');
const modeFilter = ref('All');
const statusFilter = ref('All');
const currentPage = ref(1);
const itemsPerPage = ref(10);

const pagination = ref({
  page: 1,
  limit: 10,
  total: 0,
  pages: 1,
});

let refreshInterval = null;

const form = ref({
  ip_address: '',
  reason: '',
});

const router = useRouter();

const filteredBlockedIps = computed(() => {
  return blockedIps.value.filter((ip) => {
    const ipAddress = String(
      ip.ip_address || ip.ipAddress || ''
    ).toLowerCase();

    const reason = String(ip.reason || '').toLowerCase();

    const query = searchQuery.value.toLowerCase();

    const matchesSearch =
      ipAddress.includes(query) ||
      reason.includes(query);

    const matchesMode =
      modeFilter.value === 'All' ||
      ip.mode === modeFilter.value;

    const matchesStatus =
      statusFilter.value === 'All' ||
      ip.status === statusFilter.value;

    return matchesSearch && matchesMode && matchesStatus;
  });
});

// --- Visible page buttons ---
const visiblePages = computed(() => {
  const total = pagination.value.pages || 1;
  const current = pagination.value.page || 1;
  const range = [];

  const start = Math.max(1, current - 1);
  const end = Math.min(total, current + 1);

  for (let page = start; page <= end; page += 1) {
    range.push(page);
  }

  return range;
});

// --- Helper: Format Timestamp ---
const formatTimestamp = (isoString) => {
  if (!isoString) return '—';

  const date = new Date(isoString);

  if (Number.isNaN(date.getTime())) {
    return isoString;
  }

  return new Intl.DateTimeFormat(undefined, {
    year: 'numeric',
    month: 'short',
    day: 'numeric',
    hour: '2-digit',
    minute: '2-digit',
    second: '2-digit',
  }).format(date);
};

// --- Styling Helpers ---
const modeClass = (mode) => {
  if (mode === 'Auto') return 'bg-purple-500/20 text-purple-400';
  return 'bg-blue-500/20 text-blue-400';
};

const statusClass = (status) => {
  if (status === 'Active') return 'bg-red-500/20 text-red-400';
  return 'bg-slate-500/20 text-slate-400';
};

// --- Authentication & Redirect ---
const clearAuthAndRedirect = (reason) => {
  localStorage.removeItem('access_token');
  sessionStorage.removeItem('access_token');

  error.value = `${reason} Redirecting to login...`;

  setTimeout(() => {
    router.push('/login');
  }, 1500);
};

// --- Fetch Blocked IPs using backend pagination ---
const loadBlockedIps = async () => {
  if (loading.value) return;

  loading.value = true;
  error.value = '';

  try {
    const response = await api.get('/api/v1/blocked_ips', {
      params: {
        page: currentPage.value,
        limit: itemsPerPage.value,
      },
    });

    const responseData = response.data;

    // Expected backend pagination shape:
    // { data: [...], pagination: { page, limit, total, pages } }
    blockedIps.value = Array.isArray(responseData?.data)
      ? responseData.data
      : Array.isArray(responseData)
        ? responseData
        : [];

    pagination.value = responseData?.pagination || {
      page: currentPage.value,
      limit: itemsPerPage.value,
      total: blockedIps.value.length,
      pages: 1,
    };

    currentPage.value = pagination.value.page || currentPage.value;
  } catch (err) {
    console.error('Load blocked IPs error:', err);

    if (err.response?.status === 401) {
      clearAuthAndRedirect('Your session has expired.');
    } else if (err.response?.status === 403) {
      clearAuthAndRedirect(
        'Access forbidden. Your token may be invalid or you lack permissions.'
      );
    } else if (err.code === 'ERR_NETWORK') {
      error.value = 'Cannot connect to backend. Is the server running?';
    } else {
      error.value = 'Failed to fetch blocked IPs. Please try again later.';
    }
  } finally {
    loading.value = false;
  }
};

const refreshBlockedIps = async () => {
  currentPage.value = 1;
  await loadBlockedIps();
};

const goToPage = async (page) => {
  if (
    page < 1 ||
    page > pagination.value.pages ||
    page === pagination.value.page
  ) {
    return;
  }

  currentPage.value = page;
  await loadBlockedIps();
};

// --- Submit New Block ---
const submitBlockIp = async () => {
  if (submitting.value) return;

  submitting.value = true;
  error.value = '';

  try {
    await api.post('/api/v1/blocked_ips', {
      ip_address: form.value.ip_address,
      reason: form.value.reason,
    });

    form.value = {
      ip_address: '',
      reason: '',
    };

    showForm.value = false;

    currentPage.value = 1;
    await loadBlockedIps();
  } catch (err) {
    console.error('Submit block IP error:', err);

    if (err.response?.status === 401) {
      clearAuthAndRedirect('Your session has expired.');
    } else if (err.response?.status === 403) {
      clearAuthAndRedirect(
        'Access forbidden. Your token may be invalid or you lack permissions.'
      );
    } else if (err.response?.status === 400) {
      error.value =
        err.response.data?.message ||
        err.response.data?.detail ||
        'Invalid IP address or missing data.';
    } else if (err.code === 'ERR_NETWORK') {
      error.value = 'Cannot connect to backend. Is the server running?';
    } else {
      error.value = 'Failed to block IP. Please try again later.';
    }
  } finally {
    submitting.value = false;
  }
};

// --- Polling ---
const startPolling = () => {
  if (refreshInterval) clearInterval(refreshInterval);

  refreshInterval = setInterval(() => {
    if (!loading.value && !submitting.value && !error.value?.includes('Redirecting')) {
      loadBlockedIps();
    }
  }, 30000);
};

// --- Lifecycle ---
onMounted(() => {
  loadBlockedIps();
  startPolling();
});

onUnmounted(() => {
  if (refreshInterval) clearInterval(refreshInterval);
});
</script>