<template>
  <section>
    <div class="mb-6 flex items-center justify-between">
      <div>
        <p class="text-slate-400">
          Alerts > Detected ransomware and suspicious network activities
        </p>
      </div>

      <button @click="refreshAlerts" :disabled="loading"
        class="bg-slate-800 hover:bg-slate-700 transition px-5 py-3 rounded-xl font-medium disabled:opacity-50">
        {{ loading ? "Refreshing..." : "Refresh" }}
      </button>
    </div>

    <div v-if="loading && alerts.length === 0" class="text-slate-400">
      Loading alerts...
    </div>

    <div v-else-if="error" class="bg-red-500/10 border border-red-500/30 text-red-400 p-4 rounded-xl">
      {{ error }}
    </div>

    <div v-else class="bg-slate-900 border border-slate-800 rounded-2xl p-6">
      <!-- Search + Filters -->
      <div class="mb-5 grid grid-cols-1 gap-4 lg:grid-cols-3">
        <input v-model="searchQuery" type="text" placeholder="Search source IP, destination IP, or threat..."
          class="w-full rounded-2xl border border-white/10 bg-slate-950 px-4 py-3 text-sm text-white outline-none transition focus:border-sky-400" />

        <select v-model="severityFilter"
          class="w-full rounded-2xl border border-white/10 bg-slate-950 px-4 py-3 text-sm text-white outline-none transition focus:border-sky-400">
          <option value="All">All Severities</option>
          <option value="High">High</option>
          <option value="Medium">Medium</option>
          <option value="Low">Low</option>
        </select>

        <select v-model="statusFilter"
          class="w-full rounded-2xl border border-white/10 bg-slate-950 px-4 py-3 text-sm text-white outline-none transition focus:border-sky-400">
          <option value="All">All Statuses</option>
          <option value="Blocked">Blocked</option>
          <option value="Monitoring">Monitoring</option>
          <option value="Reviewed">Reviewed</option>
          <option value="Resolved">Resolved</option>
          <option value="False Positive">False Positive</option>
        </select>
      </div>

      <div class="flex items-center justify-between mb-5">
        <div>
          <h3 class="text-lg font-semibold text-white">All Alerts</h3>
          <p class="mt-1 text-sm text-slate-400">
            Page {{ displayPagination.page }} of {{ displayPagination.pages }}
          </p>
        </div>

        <span class="px-4 py-2 rounded-xl text-sm bg-red-500/20 text-red-400">
          {{ displayedAlerts.length }} shown / {{ displayPagination.total }} total
        </span>
      </div>

      <div v-if="displayedAlerts.length === 0" class="text-slate-400">
        No alerts found.
      </div>

      <div v-else class="overflow-x-auto">
        <table class="w-full text-sm">
          <thead class="bg-slate-800 text-slate-300">
            <tr>
              <th class="text-left p-4">Time</th>
              <th class="text-left p-4">Source IP</th>
              <th class="text-left p-4">Destination IP</th>
              <th class="text-left p-4">Threat</th>
              <th class="text-left p-4">Severity</th>
              <th class="text-left p-4">Status</th>
              <th class="text-left p-4">Actions</th>
            </tr>
          </thead>

          <tbody>
            <tr v-for="alert in displayedAlerts" :key="alert.id"
              class="border-t border-slate-800 hover:bg-slate-800/50 transition">
              <td class="p-4">{{ formatTimestamp(alert.timestamp) }}</td>

              <td class="p-4 font-medium text-slate-200">
                {{ alert.sourceIp || alert.source_ip || '—' }}
              </td>

              <td class="p-4">
                <!-- {{ alert.destinationIp || alert.destination_ip || '—' }} -->
                {{ alert.destination_ip_display || alert.destinationIp || alert.destination_ip || '—' }}
              </td>

              <td class="p-4">
                {{ alert.threatType || alert.threat_type || '—' }}
              </td>

              <td class="p-4">
                <span class="px-3 py-1 rounded-full text-xs font-medium" :class="severityClass(alert.severity)">
                  {{ alert.severity || 'Unknown' }}
                </span>
              </td>

              <td class="p-4">
                <span class="px-3 py-1 rounded-full text-xs font-medium" :class="statusClass(alert.status)">
                  {{ alert.status || 'New' }}
                </span>
              </td>

              <td class="p-4">
                <div class="flex flex-wrap gap-2">
                  
                  <!-- <button @click="openStatusPopup(alert, 'Reviewed')" :disabled="updatingAlertId === alert.id"
                    class="px-3 py-2 rounded-lg text-xs bg-blue-600 hover:bg-blue-700 transition disabled:opacity-50">
                    Review
                  </button>

                  <button @click="openStatusPopup(alert, 'Resolved')" :disabled="updatingAlertId === alert.id"
                    class="px-3 py-2 rounded-lg text-xs bg-green-600 hover:bg-green-700 transition disabled:opacity-50">
                    Resolve
                  </button> -->

                  <button @click="openStatusPopup(alert, 'False Positive')" :disabled="updatingAlertId === alert.id"
                    class="px-3 py-2 rounded-lg text-xs bg-slate-700 hover:bg-slate-600 transition disabled:opacity-50">
                    Mark as False +
                  </button>
                </div>
              </td>
            </tr>
          </tbody>
        </table>

        <div v-if="displayPagination.pages > 1"
          class="mt-5 flex flex-col gap-3 sm:flex-row sm:items-center sm:justify-between">
          <p class="text-sm text-slate-400">
            Showing {{ displayedAlerts.length }} records on page
            {{ displayPagination.page }} of {{ displayPagination.pages }} · Total
            {{ displayPagination.total }}
          </p>

          <div class="flex items-center gap-2">
            <button @click="goToPage(displayPagination.page - 1)" :disabled="displayPagination.page === 1 || loading"
              class="rounded-xl border border-white/10 bg-white/[0.04] px-4 py-2 text-sm text-slate-300 transition hover:bg-white/[0.08] disabled:opacity-40">
              Previous
            </button>

            <button v-for="page in visiblePages" :key="page" @click="goToPage(page)" :disabled="loading" :class="[
              'rounded-xl border px-4 py-2 text-sm transition disabled:opacity-40',
              page === displayPagination.page
                ? 'border-sky-400/30 bg-sky-400 text-slate-950'
                : 'border-white/10 bg-white/[0.04] text-slate-300 hover:bg-white/[0.08]'
            ]">
              {{ page }}
            </button>

            <button @click="goToPage(displayPagination.page + 1)"
              :disabled="displayPagination.page === displayPagination.pages || loading"
              class="rounded-xl border border-white/10 bg-white/[0.04] px-4 py-2 text-sm text-slate-300 transition hover:bg-white/[0.08] disabled:opacity-40">
              Next
            </button>
          </div>
        </div>
      </div>
    </div>

    <div v-if="showStatusPopup" class="fixed inset-0 z-50 flex items-center justify-center bg-black/70 px-4">
      <div class="w-full max-w-lg rounded-2xl border border-slate-700 bg-slate-950 p-6 shadow-2xl">
        <h3 class="text-xl font-bold text-white mb-2">
          Confirm Alert Action
        </h3>

        <p class="text-sm text-slate-400 mb-5">
          Are you sure you want to mark this alert as
          <span class="font-semibold text-white">{{ pendingStatus }}</span>?
        </p>

        <div class="mb-5 rounded-xl border border-slate-800 bg-slate-900 p-4 text-sm">
          <div class="mb-2 flex justify-between gap-4">
            <span class="text-slate-400">Source IP</span>
            <span class="text-white">
              {{ selectedAlert?.sourceIp || selectedAlert?.source_ip || '—' }}
            </span>
          </div>

          <div class="mb-2 flex justify-between gap-4">
            <span class="text-slate-400">Destination IP</span>
            <span class="text-white">
              <!-- {{ selectedAlert?.destinationIp || selectedAlert?.destination_ip || '—' }} -->
              {{ selectedAlert?.destination_ip_display || selectedAlert?.destinationIp || selectedAlert?.destination_ip || '—' }}
            </span>
          </div>

          <div class="mb-2 flex justify-between gap-4">
            <span class="text-slate-400">Severity</span>
            <span class="text-white">
              {{ selectedAlert?.severity || '—' }}
            </span>
          </div>

          <div class="flex justify-between gap-4">
            <span class="text-slate-400">Current Status</span>
            <span class="text-white">
              {{ selectedAlert?.status || 'New' }}
            </span>
          </div>
        </div>

        <div v-if="pendingStatus === 'False Positive'"
          class="mb-5 rounded-xl border border-yellow-500/30 bg-yellow-500/10 p-4 text-sm text-yellow-300">
          This may trigger gateway unblock logic if the IP was previously blocked.
        </div>

        <div class="flex justify-end gap-3">
          <button @click="closeStatusPopup" :disabled="updatingAlertId"
            class="rounded-xl border border-slate-700 px-4 py-2 text-sm text-slate-300 hover:bg-slate-800 disabled:opacity-50">
            Cancel
          </button>

          <button @click="confirmStatusUpdate" :disabled="updatingAlertId"
            class="rounded-xl bg-sky-600 px-4 py-2 text-sm font-medium text-white hover:bg-sky-700 disabled:opacity-50">
            {{ updatingAlertId ? 'Processing...' : 'Confirm' }}
          </button>
        </div>
      </div>
    </div>

    <div v-if="successMessage" class="mt-4 bg-green-500/10 border border-green-500/30 text-green-400 p-4 rounded-xl">
      {{ successMessage }}
    </div>
  </section>
</template>

<script setup>
import { computed, onMounted, onUnmounted, ref, watch } from 'vue';
import { useRouter } from 'vue-router';
import api from '../services/api';

const loading = ref(false);
const error = ref('');
const successMessage = ref('');

const alerts = ref([]);       // current page only
const allAlerts = ref([]);    // all pages, used during search/filter

const updatingAlertId = ref(null);

const searchQuery = ref('');
const severityFilter = ref('All');
const statusFilter = ref('All');

const currentPage = ref(1);
const itemsPerPage = ref(10);

const showStatusPopup = ref(false);
const selectedAlert = ref(null);
const pendingStatus = ref('');

const pagination = ref({
  page: 1,
  limit: 10,
  total: 0,
  pages: 1,
});

let refreshInterval = null;
let successTimeout = null;
let filterDebounce = null;

const router = useRouter();

const hasActiveFilters = computed(() => {
  return (
    searchQuery.value.trim() !== '' ||
    severityFilter.value !== 'All' ||
    statusFilter.value !== 'All'
  );
});

const filterAlertList = (list) => {
  return list.filter((alert) => {
    const sourceIp = String(alert.sourceIp || alert.source_ip || '').toLowerCase();
    const destinationIp = String(alert.destinationIp || alert.destination_ip || '').toLowerCase();
    const threat = String(alert.threatType || alert.threat_type || '').toLowerCase();
    const severity = String(alert.severity || '');
    const status = String(alert.status || '');

    const query = searchQuery.value.trim().toLowerCase();

    const matchesSearch =
      query === '' ||
      sourceIp.includes(query) ||
      destinationIp.includes(query) ||
      threat.includes(query);

    const matchesSeverity =
      severityFilter.value === 'All' ||
      severity === severityFilter.value;

    const matchesStatus =
      statusFilter.value === 'All' ||
      status === statusFilter.value;

    return matchesSearch && matchesSeverity && matchesStatus;
  });
};

const filteredAlerts = computed(() => {
  if (!hasActiveFilters.value) {
    return alerts.value;
  }

  return filterAlertList(allAlerts.value);
});

const displayedAlerts = computed(() => {
  if (!hasActiveFilters.value) {
    return alerts.value;
  }

  const start = (currentPage.value - 1) * itemsPerPage.value;
  const end = start + itemsPerPage.value;

  return filteredAlerts.value.slice(start, end);
});

const displayPagination = computed(() => {
  if (!hasActiveFilters.value) {
    return pagination.value;
  }

  const total = filteredAlerts.value.length;
  const pages = Math.max(Math.ceil(total / itemsPerPage.value), 1);

  return {
    page: currentPage.value,
    limit: itemsPerPage.value,
    total,
    pages,
  };
});

const visiblePages = computed(() => {
  const total = displayPagination.value.pages || 1;
  const current = displayPagination.value.page || 1;
  const range = [];

  const start = Math.max(1, current - 1);
  const end = Math.min(total, current + 1);

  for (let page = start; page <= end; page += 1) {
    range.push(page);
  }

  return range;
});

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

const severityClass = (severity) => {
  if (severity === 'High') return 'bg-red-500/20 text-red-400';
  if (severity === 'Medium') return 'bg-yellow-500/20 text-yellow-400';
  if (severity === 'Low') return 'bg-green-500/20 text-green-400';
  return 'bg-gray-500/20 text-gray-400';
};

const statusClass = (status) => {
  const classes = {
    Blocked: 'bg-red-500/20 text-red-400',
    Monitoring: 'bg-yellow-500/20 text-yellow-400',
    Reviewed: 'bg-blue-500/20 text-blue-400',
    Resolved: 'bg-green-500/20 text-green-400',
    'False Positive': 'bg-slate-500/20 text-slate-300',
  };

  return classes[status] || 'bg-slate-500/20 text-slate-400';
};

const clearAuthAndRedirect = (reason) => {
  localStorage.removeItem('access_token');
  sessionStorage.removeItem('access_token');

  error.value = `${reason} Redirecting to login...`;

  setTimeout(() => {
    router.push('/login');
  }, 1500);
};

const normalizeAlertsResponse = (responseData) => {
  return {
    alerts: Array.isArray(responseData?.data)
      ? responseData.data
      : Array.isArray(responseData)
        ? responseData
        : [],

    pagination: responseData?.pagination || {
      page: currentPage.value,
      limit: itemsPerPage.value,
      total: 0,
      pages: 1,
    },
  };
};

const fetchAlertPage = async (page) => {
  const response = await api.get('/api/v1/alerts', {
    params: {
      page,
      limit: itemsPerPage.value,
    },
  });

  return normalizeAlertsResponse(response.data);
};

const loadCurrentPageAlerts = async () => {
  const result = await fetchAlertPage(currentPage.value);

  alerts.value = result.alerts;

  pagination.value = result.pagination || {
    page: currentPage.value,
    limit: itemsPerPage.value,
    total: alerts.value.length,
    pages: 1,
  };

  currentPage.value = pagination.value.page || currentPage.value;
};

const loadAllAlertsForSearch = async () => {
  const firstResult = await fetchAlertPage(1);

  let collectedAlerts = [...firstResult.alerts];

  const totalPages = firstResult.pagination?.pages || 1;

  for (let page = 2; page <= totalPages; page += 1) {
    const result = await fetchAlertPage(page);
    collectedAlerts = collectedAlerts.concat(result.alerts);
  }

  allAlerts.value = collectedAlerts;

  pagination.value = {
    page: currentPage.value,
    limit: itemsPerPage.value,
    total: collectedAlerts.length,
    pages: Math.max(Math.ceil(collectedAlerts.length / itemsPerPage.value), 1),
  };

  const maxPage = displayPagination.value.pages;

  if (currentPage.value > maxPage) {
    currentPage.value = maxPage;
  }
};

const loadAlerts = async () => {
  if (loading.value) return;

  loading.value = true;
  error.value = '';

  try {
    if (hasActiveFilters.value) {
      await loadAllAlertsForSearch();
    } else {
      allAlerts.value = [];
      await loadCurrentPageAlerts();
    }
  } catch (err) {
    console.error('Load alerts error:', err);

    if (err.response?.status === 401) {
      clearAuthAndRedirect('Your session has expired.');
    } else if (err.response?.status === 403) {
      clearAuthAndRedirect(
        'Access forbidden. Your token may be invalid or you lack permissions.'
      );
    } else if (err.code === 'ERR_NETWORK') {
      error.value = 'Cannot connect to backend. Is the server running?';
    } else {
      error.value = 'Failed to fetch alerts. Please try again later.';
    }
  } finally {
    loading.value = false;
  }
};

const refreshAlerts = async () => {
  currentPage.value = 1;
  await loadAlerts();
};

const goToPage = async (page) => {
  if (
    page < 1 ||
    page > displayPagination.value.pages ||
    page === displayPagination.value.page
  ) {
    return;
  }

  currentPage.value = page;

  if (!hasActiveFilters.value) {
    await loadAlerts();
  }
};

const replaceAlertInLists = (updatedAlert) => {
  alerts.value = alerts.value.map((alert) =>
    alert.id === updatedAlert.id ? updatedAlert : alert
  );

  allAlerts.value = allAlerts.value.map((alert) =>
    alert.id === updatedAlert.id ? updatedAlert : alert
  );
};


const openStatusPopup = (alert, status) => {
  selectedAlert.value = alert;
  pendingStatus.value = status;
  showStatusPopup.value = true;
};

const closeStatusPopup = () => {
  if (updatingAlertId.value) return;

  showStatusPopup.value = false;
  selectedAlert.value = null;
  pendingStatus.value = '';
};

const confirmStatusUpdate = async () => {
  if (!selectedAlert.value || !pendingStatus.value) return;

  await updateStatus(selectedAlert.value.id, pendingStatus.value);
  closeStatusPopup();
};


const updateStatus = async (alertId, status) => {
  if (updatingAlertId.value === alertId) return;

  updatingAlertId.value = alertId;
  error.value = '';
  successMessage.value = '';

  if (successTimeout) clearTimeout(successTimeout);

  try {
    const response = await api.patch(`/api/v1/alerts/${alertId}/status`, {
      status,
    });

    const updatedAlert = response.data?.alert || response.data;
    const gatewayUnblockResult = response.data?.gateway_unblock_result || null;

    replaceAlertInLists(updatedAlert);


    if (status === 'False Positive' && gatewayUnblockResult) {
      const results = Array.isArray(gatewayUnblockResult)
        ? gatewayUnblockResult
        : [gatewayUnblockResult];

      const successful = results.filter((item) => item.success);
      const failed = results.filter((item) => !item.success);

      if (successful.length > 0 && failed.length === 0) {
        successMessage.value =
          `Alert marked as False Positive and ${successful.length} IP(s) unblocked successfully.`;
      } else if (successful.length > 0 && failed.length > 0) {
        successMessage.value =
          `Alert marked as False Positive. ${successful.length} IP(s) unblocked, ${failed.length} failed.`;
      } else {
        const firstError =
          failed[0]?.response?.error ||
          failed[0]?.response?.message ||
          'Gateway unblock failed.';

        successMessage.value =
          `Alert marked as False Positive, but gateway unblock failed: ${firstError}`;
      }
    } else {
      successMessage.value = `Alert marked as ${status}.`;
    }

    successTimeout = setTimeout(() => {
      successMessage.value = '';
    }, 3000);
  } catch (err) {
    console.error('Update status error:', err);

    if (err.response?.status === 401) {
      clearAuthAndRedirect('Your session has expired.');
    } else if (err.response?.status === 403) {
      clearAuthAndRedirect(
        'Access forbidden. Your token may be invalid or you lack permissions.'
      );
    } else if (err.code === 'ERR_NETWORK') {
      error.value = 'Cannot connect to backend. Is the server running?';
    } else {
      const detail = err.response?.data?.detail || err.message;
      error.value = `Failed to update alert status: ${detail}`;
    }
  } finally {
    updatingAlertId.value = null;
  }
};

watch(
  [searchQuery, severityFilter, statusFilter],
  () => {
    if (filterDebounce) clearTimeout(filterDebounce);

    filterDebounce = setTimeout(async () => {
      currentPage.value = 1;
      await loadAlerts();
    }, 350);
  }
);

const startPolling = () => {
  if (refreshInterval) clearInterval(refreshInterval);

  refreshInterval = setInterval(() => {
    if (
      !loading.value &&
      !updatingAlertId.value &&
      !error.value?.includes('Redirecting')
    ) {
      loadAlerts();
    }
  }, 30000);
};

onMounted(() => {
  loadAlerts();
  startPolling();
});

onUnmounted(() => {
  if (refreshInterval) clearInterval(refreshInterval);
  if (successTimeout) clearTimeout(successTimeout);
  if (filterDebounce) clearTimeout(filterDebounce);
});
</script>