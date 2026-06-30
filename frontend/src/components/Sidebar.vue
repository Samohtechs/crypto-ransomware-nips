<template>
  <aside
    :class="[
      'sticky top-0 h-screen bg-[#050816] border-r border-white/10 flex flex-col transition-all duration-300 ease-in-out overflow-hidden',
      collapsed ? 'w-[72px] px-3 py-5' : 'w-72 p-5'
    ]"
  >
    <!-- Brand -->
    <div :class="['mb-8 flex flex-col', collapsed ? 'items-center' : '']">
      <div :class="['flex items-center', collapsed ? 'justify-center' : 'gap-3']">
        <div class="flex-shrink-0 flex h-11 w-11 items-center justify-center rounded-2xl bg-sky-500/15 border border-sky-400/30">
          <svg viewBox="0 0 24 24" class="h-5 w-5 text-sky-300" fill="none" stroke="currentColor" stroke-width="2">
            <path stroke-linecap="round" stroke-linejoin="round" d="M9 12.75L11.25 15 15 9.75m-3-7.036A11.959 11.959 0 013.598 6 11.99 11.99 0 003 9.749c0 5.592 3.824 10.29 9 11.623 5.176-1.332 9-6.03 9-11.622 0-1.31-.21-2.571-.598-3.751h-.152c-3.196 0-6.1-1.248-8.25-3.285z" />
          </svg>
        </div>
        <Transition name="fade">
          <div v-if="!collapsed">
            <h1 class="text-xl font-bold tracking-tight text-white">AI-NIPS</h1>
            <p class="text-xs text-slate-400">Crypto-Ransomware Defense</p>
          </div>
        </Transition>
      </div>

      
    </div>

    <!-- Navigation -->
    <nav class="space-y-1 flex-1 overflow-y-auto pr-1">
      <RouterLink
        v-for="item in links"
        :key="item.path"
        :to="item.path"
        :title="collapsed ? item.name : ''"
        :class="[
          'group flex items-center rounded-2xl py-3 text-sm font-medium transition-all duration-200',
          collapsed ? 'justify-center px-3' : 'gap-3 px-4',
          isActive(item.path)
            ? 'bg-sky-400 text-slate-950 shadow-lg shadow-sky-500/20'
            : 'text-slate-400 hover:bg-white/[0.04] hover:text-white'
        ]"
      >
        <component
          :is="item.icon"
          :class="['h-5 w-5 flex-shrink-0', isActive(item.path) ? 'text-slate-950' : 'text-slate-400 group-hover:text-white']"
        />
        <Transition name="fade">
          <span v-if="!collapsed" class="flex-1">{{ item.name }}</span>
        </Transition>
        <span
          v-if="isActive(item.path) && !collapsed"
          class="h-2 w-2 rounded-full bg-slate-950 flex-shrink-0"
        ></span>
      </RouterLink>
    </nav>

    <!-- Footer -->
    <div :class="['mt-6 border-t border-white/10 pt-5 flex flex-col gap-2', collapsed ? 'items-center' : '']">
      
     

      <!-- Role Badge -->
<div v-if="!collapsed" class="rounded-2xl border border-white/10 bg-white/[0.03] px-4 py-3 flex flex-row items-center justify-center gap-4">
  <p class="text-xs text-slate-500">Logged in as</p>
  <div class="flex items-center gap-2">
    <span class="h-2 w-2 rounded-full" :class="roleDotClass"></span>
    <span class="text-xs font-semibold uppercase tracking-wide" :class="roleTextClass">
      {{ userRole }}
    </span>
  </div>
</div>

       

      <!-- Hii itakuwa Manage User -->
      <RouterLink
        v-if="userRole === 'admin'"
  to="/register"
        :title="collapsed ? 'Manage User' : ''"
        :class="[
          'flex items-center justify-center rounded-2xl border border-sky-400/20 bg-sky-400/10 text-sm font-medium text-sky-300 transition hover:bg-sky-500 hover:text-white hover:border-sky-500',
          collapsed ? 'h-11 w-11' : 'w-full gap-3 px-4 py-3',
          isActive('/register') ? 'bg-sky-500 text-white border-sky-500' : ''
        ]"
      >
        <!-- Plus icon -->
        <svg class="h-5 w-5 flex-shrink-0" fill="none" stroke="currentColor" stroke-width="2" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" d="M18 9v3m0 0v3m0-3h3m-3 0h-3m-2-5a4 4 0 11-8 0 4 4 0 018 0zM3 20a6 6 0 0112 0v1H3v-1z" />
        </svg>
        <Transition name="fade">
          <span v-if="!collapsed">Manage Users</span>
        </Transition>
      </RouterLink>

      <!-- Logout -->
      <button
        @click="handleLogout"
        :title="collapsed ? 'Logout' : ''"
        :class="[
          'flex items-center justify-center rounded-2xl border border-red-400/20 bg-red-400/10 text-sm font-medium text-red-300 transition hover:bg-red-500 hover:text-white hover:border-red-500',
          collapsed ? 'h-11 w-11' : 'w-full gap-3 px-4 py-3'
        ]"
      >
        <!-- Logout icon -->
        <svg class="h-5 w-5 flex-shrink-0" fill="none" stroke="currentColor" stroke-width="2" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" d="M15.75 9V5.25A2.25 2.25 0 0013.5 3h-6a2.25 2.25 0 00-2.25 2.25v13.5A2.25 2.25 0 007.5 21h6a2.25 2.25 0 002.25-2.25V15M12 9l-3 3m0 0l3 3m-3-3h12.75" />
        </svg>
        <Transition name="fade">
          <span v-if="!collapsed">Logout</span>
        </Transition>
      </button>
    </div>
  </aside>
</template>

<script setup>
import { computed, h, onMounted, onUnmounted, ref } from 'vue';
import { useRoute, useRouter } from 'vue-router';

const collapsed = defineModel('collapsed', { default: false });
const route = useRoute();
const router = useRouter();

const now = ref(new Date());
let clockTimer = null;

const currentDate = computed(() => {
  return new Intl.DateTimeFormat('en-GB', {
    day: '2-digit',
    month: 'short',
    year: 'numeric',
  }).format(now.value);
});



const currentTime = computed(() => {
  return new Intl.DateTimeFormat('en-GB', {
    hour: '2-digit',
    minute: '2-digit',
    second: '2-digit',
    hour12: false,
  }).format(now.value);
});

onMounted(() => {
  clockTimer = setInterval(() => {
    now.value = new Date();
  }, 1000);
});

onUnmounted(() => {
  if (clockTimer) {
    clearInterval(clockTimer);
  }
});

// Inline SVG icon components
const IconDashboard = { render: () => h('svg', { fill: 'none', stroke: 'currentColor', 'stroke-width': '2', viewBox: '0 0 24 24' }, [
  h('path', { 'stroke-linecap': 'round', 'stroke-linejoin': 'round', d: 'M3.75 6A2.25 2.25 0 016 3.75h2.25A2.25 2.25 0 0110.5 6v2.25a2.25 2.25 0 01-2.25 2.25H6a2.25 2.25 0 01-2.25-2.25V6zM3.75 15.75A2.25 2.25 0 016 13.5h2.25a2.25 2.25 0 012.25 2.25V18A2.25 2.25 0 018.25 20.25H6A2.25 2.25 0 013.75 18v-2.25zM13.5 6a2.25 2.25 0 012.25-2.25H18A2.25 2.25 0 0120.25 6v2.25A2.25 2.25 0 0118 10.5h-2.25a2.25 2.25 0 01-2.25-2.25V6zM13.5 15.75a2.25 2.25 0 012.25-2.25H18a2.25 2.25 0 012.25 2.25V18A2.25 2.25 0 0118 20.25h-2.25A2.25 2.25 0 0113.5 18v-2.25z' })
]) };

const IconAlerts = { render: () => h('svg', { fill: 'none', stroke: 'currentColor', 'stroke-width': '2', viewBox: '0 0 24 24' }, [
  h('path', { 'stroke-linecap': 'round', 'stroke-linejoin': 'round', d: 'M14.857 17.082a23.848 23.848 0 005.454-1.31A8.967 8.967 0 0118 9.75v-.7V9A6 6 0 006 9v.75a8.967 8.967 0 01-2.312 6.022c1.733.64 3.56 1.085 5.455 1.31m5.714 0a24.255 24.255 0 01-5.714 0m5.714 0a3 3 0 11-5.714 0' })
]) };

const IconTraffic = { render: () => h('svg', { fill: 'none', stroke: 'currentColor', 'stroke-width': '2', viewBox: '0 0 24 24' }, [
  h('path', { 'stroke-linecap': 'round', 'stroke-linejoin': 'round', d: 'M3 13.125C3 12.504 3.504 12 4.125 12h2.25c.621 0 1.125.504 1.125 1.125v6.75C7.5 20.496 6.996 21 6.375 21h-2.25A1.125 1.125 0 013 19.875v-6.75zM9.75 8.625c0-.621.504-1.125 1.125-1.125h2.25c.621 0 1.125.504 1.125 1.125v11.25c0 .621-.504 1.125-1.125 1.125h-2.25a1.125 1.125 0 01-1.125-1.125V8.625zM16.5 4.125c0-.621.504-1.125 1.125-1.125h2.25C20.496 3 21 3.504 21 4.125v15.75c0 .621-.504 1.125-1.125 1.125h-2.25a1.125 1.125 0 01-1.125-1.125V4.125z' })
]) };

const IconBlocked = { render: () => h('svg', { fill: 'none', stroke: 'currentColor', 'stroke-width': '2', viewBox: '0 0 24 24' }, [
  h('path', { 'stroke-linecap': 'round', 'stroke-linejoin': 'round', d: 'M18.364 18.364A9 9 0 005.636 5.636m12.728 12.728A9 9 0 015.636 5.636m12.728 12.728L5.636 5.636' })
]) };

const IconModel = { render: () => h('svg', { fill: 'none', stroke: 'currentColor', 'stroke-width': '2', viewBox: '0 0 24 24' }, [
  h('path', { 'stroke-linecap': 'round', 'stroke-linejoin': 'round', d: 'M8.25 3v1.5M4.5 8.25H3m18 0h-1.5M4.5 12H3m18 0h-1.5m-15 3.75H3m18 0h-1.5M8.25 19.5V21M12 3v1.5m0 15V21m3.75-18v1.5m0 15V21m-9-1.5h10.5a2.25 2.25 0 002.25-2.25V6.75a2.25 2.25 0 00-2.25-2.25H6.75A2.25 2.25 0 004.5 6.75v10.5a2.25 2.25 0 002.25 2.25zm.75-12h9v9h-9v-9z' })
]) };

const IconSimulator = { render: () => h('svg', { fill: 'none', stroke: 'currentColor', 'stroke-width': '2', viewBox: '0 0 24 24' }, [
  h('path', { 'stroke-linecap': 'round', 'stroke-linejoin': 'round', d: 'M3.75 13.5l10.5-11.25L12 10.5h8.25L9.75 21.75 12 13.5H3.75z' })
]) };



const allLinks = [
  {
    name: 'Dashboard',
    path: '/dashboard',
    icon: IconDashboard,
    roles: ['admin', 'analyst', 'viewer'],
  },

  {
    name: 'Alerts',
    path: '/alerts',
    icon: IconAlerts,
    roles: ['admin', 'analyst'],
  },

  {
    name: 'Traffic Monitor',
    path: '/traffic',
    icon: IconTraffic,
    roles: ['admin', 'analyst', 'viewer'],
  },

  {
    name: 'Blocked IPs',
    path: '/blocked_ips',
    icon: IconBlocked,
    roles: ['admin', 'analyst'],
  },

  {
    name: 'Model Status',
    path: '/model_status',
    icon: IconModel,
    roles: ['admin', 'analyst', 'viewer'],
  },

  // {
  //   name: 'Attack Simulator',
  //   path: '/simulator',
  //   icon: IconSimulator,
  //   roles: ['admin', 'analyst'],
  // },
];

const links = computed(() => {
  return allLinks.filter((item) =>
    item.roles.includes(userRole.value)
  );
});

const username = computed(() => {
  try {
    const token = localStorage.getItem('access_token');
    if (!token) return 'Administrator';
    const payload = JSON.parse(atob(token.split('.')[1]));
    return payload.sub || payload.username || 'Administrator';
  } catch {
    return 'Administrator';
  }
});

const isActive = (path) => route.path === path;

const handleLogout = () => {
  localStorage.removeItem('access_token');
  sessionStorage.removeItem('access_token');
  localStorage.removeItem('user_role');
  sessionStorage.setItem('authWarning', 'You have been logged out.');
  router.push('/login');
};

const userRole = computed(() => {
  return localStorage.getItem('user_role') || 'viewer';
});

const roleTextClass = computed(() => {
  if (userRole.value === 'admin') return 'text-red-300';
  if (userRole.value === 'analyst') return 'text-sky-300';
  return 'text-emerald-300';
});

const roleDotClass = computed(() => {
  if (userRole.value === 'admin') return 'bg-red-400';
  if (userRole.value === 'analyst') return 'bg-sky-400';
  return 'bg-emerald-400';
});

</script>
<style scoped>
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.2s ease;
}
.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}
</style>