<template>
  <div class="app-shell" :class="{ 'sidebar-collapsed': sidebarCollapsed }">
    <nav class="sidebar" :class="{ collapsed: sidebarCollapsed }">
      <div class="sidebar-brand">
        {{ t('nav.companyName') }}
      </div>
      <button class="sidebar-toggle" @click="toggleSidebar" :title="sidebarCollapsed ? 'Expand sidebar' : 'Collapse sidebar'">
        <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.75" stroke-linecap="round" stroke-linejoin="round">
          <path v-if="!sidebarCollapsed" d="M10 3L6 8l4 5"/>
          <path v-else d="M6 3l4 5-4 5"/>
        </svg>
      </button>
      <div class="sidebar-nav">
        <router-link to="/" class="nav-item" exact-active-class="router-link-active" data-tooltip="Overview">
          <svg class="nav-icon" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.75" stroke-linecap="round" stroke-linejoin="round">
            <rect x="1" y="1" width="6" height="6" rx="1"/>
            <rect x="9" y="1" width="6" height="6" rx="1"/>
            <rect x="1" y="9" width="6" height="6" rx="1"/>
            <rect x="9" y="9" width="6" height="6" rx="1"/>
          </svg>
          <span class="nav-label">{{ t('nav.overview') }}</span>
        </router-link>
        <router-link to="/inventory" class="nav-item" data-tooltip="Inventory">
          <svg class="nav-icon" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.75" stroke-linecap="round" stroke-linejoin="round">
            <rect x="2" y="3" width="12" height="10" rx="1"/>
            <path d="M5 3V2M11 3V2"/>
            <path d="M2 7h12"/>
            <path d="M5 10h2M9 10h2"/>
          </svg>
          <span class="nav-label">{{ t('nav.inventory') }}</span>
        </router-link>
        <router-link to="/orders" class="nav-item" data-tooltip="Orders">
          <svg class="nav-icon" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.75" stroke-linecap="round" stroke-linejoin="round">
            <path d="M11 2H5a1 1 0 00-1 1v10a1 1 0 001 1h6a1 1 0 001-1V3a1 1 0 00-1-1z"/>
            <path d="M6 6h4M6 9h4M6 12h2"/>
          </svg>
          <span class="nav-label">{{ t('nav.orders') }}</span>
        </router-link>
        <router-link to="/spending" class="nav-item" data-tooltip="Finance">
          <svg class="nav-icon" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.75" stroke-linecap="round" stroke-linejoin="round">
            <rect x="1" y="4" width="14" height="9" rx="1.5"/>
            <path d="M1 7h14"/>
            <path d="M4 11h2"/>
          </svg>
          <span class="nav-label">{{ t('nav.finance') }}</span>
        </router-link>
        <router-link to="/demand" class="nav-item" data-tooltip="Demand Forecast">
          <svg class="nav-icon" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.75" stroke-linecap="round" stroke-linejoin="round">
            <path d="M1 12L5 8l3 2 4-5 3 2"/>
            <path d="M11 5h3v3"/>
          </svg>
          <span class="nav-label">{{ t('nav.demandForecast') }}</span>
        </router-link>
        <router-link to="/reports" class="nav-item" data-tooltip="Reports">
          <svg class="nav-icon" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.75" stroke-linecap="round" stroke-linejoin="round">
            <rect x="2" y="2" width="12" height="12" rx="1"/>
            <path d="M5 9v3M8 6v6M11 4v8"/>
          </svg>
          <span class="nav-label">Reports</span>
        </router-link>
      </div>
      <div class="sidebar-footer">
        <LanguageSwitcher :collapsed="sidebarCollapsed" />
        <ProfileMenu
          :collapsed="sidebarCollapsed"
          @show-profile-details="showProfileDetails = true"
          @show-tasks="showTasks = true"
        />
      </div>
    </nav>

    <div class="main-area">
      <FilterBar />
      <main class="page-content">
        <router-view />
      </main>
    </div>

    <ProfileDetailsModal
      :is-open="showProfileDetails"
      @close="showProfileDetails = false"
    />

    <TasksModal
      :is-open="showTasks"
      :tasks="tasks"
      @close="showTasks = false"
      @add-task="addTask"
      @delete-task="deleteTask"
      @toggle-task="toggleTask"
    />
  </div>
</template>

<script>
import { ref, onMounted, onUnmounted, computed } from 'vue'
import { api } from './api'
import { useAuth } from './composables/useAuth'
import { useI18n } from './composables/useI18n'
import FilterBar from './components/FilterBar.vue'
import ProfileMenu from './components/ProfileMenu.vue'
import ProfileDetailsModal from './components/ProfileDetailsModal.vue'
import TasksModal from './components/TasksModal.vue'
import LanguageSwitcher from './components/LanguageSwitcher.vue'

export default {
  name: 'App',
  components: {
    FilterBar,
    ProfileMenu,
    ProfileDetailsModal,
    TasksModal,
    LanguageSwitcher
  },
  setup() {
    const { currentUser } = useAuth()
    const { t } = useI18n()
    const showProfileDetails = ref(false)
    const showTasks = ref(false)
    const apiTasks = ref([])

    // Merge mock tasks from currentUser with API tasks
    const tasks = computed(() => {
      return [...currentUser.value.tasks, ...apiTasks.value]
    })

    const loadTasks = async () => {
      try {
        apiTasks.value = await api.getTasks()
      } catch (err) {
        console.error('Failed to load tasks:', err)
      }
    }

    const addTask = async (taskData) => {
      try {
        const newTask = await api.createTask(taskData)
        // Add new task to the beginning of the array
        apiTasks.value.unshift(newTask)
      } catch (err) {
        console.error('Failed to add task:', err)
      }
    }

    const deleteTask = async (taskId) => {
      try {
        // Check if it's a mock task (from currentUser)
        const isMockTask = currentUser.value.tasks.some(t => t.id === taskId)

        if (isMockTask) {
          // Remove from mock tasks
          const index = currentUser.value.tasks.findIndex(t => t.id === taskId)
          if (index !== -1) {
            currentUser.value.tasks.splice(index, 1)
          }
        } else {
          // Remove from API tasks
          await api.deleteTask(taskId)
          apiTasks.value = apiTasks.value.filter(t => t.id !== taskId)
        }
      } catch (err) {
        console.error('Failed to delete task:', err)
      }
    }

    const toggleTask = async (taskId) => {
      try {
        // Check if it's a mock task (from currentUser)
        const mockTask = currentUser.value.tasks.find(t => t.id === taskId)

        if (mockTask) {
          // Toggle mock task status
          mockTask.status = mockTask.status === 'pending' ? 'completed' : 'pending'
        } else {
          // Toggle API task
          const updatedTask = await api.toggleTask(taskId)
          const index = apiTasks.value.findIndex(t => t.id === taskId)
          if (index !== -1) {
            apiTasks.value[index] = updatedTask
          }
        }
      } catch (err) {
        console.error('Failed to toggle task:', err)
      }
    }

    const sidebarCollapsed = ref(window.innerWidth < 1100)

    const toggleSidebar = () => {
      sidebarCollapsed.value = !sidebarCollapsed.value
    }

    const handleResize = () => {
      if (window.innerWidth < 1100) {
        sidebarCollapsed.value = true
      }
    }

    onMounted(() => {
      loadTasks()
      window.addEventListener('resize', handleResize)
    })

    onUnmounted(() => {
      window.removeEventListener('resize', handleResize)
    })

    return {
      t,
      showProfileDetails,
      showTasks,
      tasks,
      addTask,
      deleteTask,
      toggleTask,
      sidebarCollapsed,
      toggleSidebar
    }
  }
}
</script>

<style>
:root {
  --bg-app:       #0f172a;
  --bg-sidebar:   #0f172a;
  --bg-surface:   #1e293b;
  --bg-surface-2: #263248;
  --bg-hover:     rgba(255, 255, 255, 0.04);
  --border:       #1e293b;
  --border-light: #334155;
  --text-primary:   #f1f5f9;
  --text-secondary: #94a3b8;
  --text-muted:     #64748b;
  --accent:        #3b82f6;
  --accent-hover:  #2563eb;
  --accent-subtle: rgba(59, 130, 246, 0.12);
  --status-green: #22c55e;
  --status-amber: #f59e0b;
  --status-red:   #ef4444;
  --status-blue:  #3b82f6;
  --space-1:  4px;  --space-2:  8px;  --space-3:  12px;
  --space-4:  16px; --space-5:  20px; --space-6:  24px;
  --space-8:  32px; --space-10: 40px; --space-12: 48px;
  --radius-sm: 6px; --radius-md: 8px; --radius-lg: 12px;
  --sidebar-width: 220px;
  --sidebar-collapsed-width: 56px;
  --transition: 150ms ease;
}

*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', system-ui, sans-serif;
  background: var(--bg-app);
  color: var(--text-primary);
  font-size: 14px;
  line-height: 1.5;
  -webkit-font-smoothing: antialiased;
}
a { color: inherit; text-decoration: none; }
button { cursor: pointer; }

.app-shell {
  display: grid;
  grid-template-columns: var(--sidebar-width) 1fr;
  min-height: 100vh;
  background: var(--bg-app);
  transition: grid-template-columns 250ms ease;
}

.app-shell.sidebar-collapsed {
  grid-template-columns: var(--sidebar-collapsed-width) 1fr;
}

.main-area {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
  overflow-x: hidden;
  background: var(--bg-app);
}

.page-content {
  flex: 1;
  padding: var(--space-8) var(--space-10);
}

.sidebar {
  width: var(--sidebar-width);
  background: var(--bg-sidebar);
  border-right: 1px solid var(--border-light);
  display: flex;
  flex-direction: column;
  padding: var(--space-5) var(--space-3);
  position: sticky;
  top: 0;
  height: 100vh;
  overflow-y: auto;
  overflow-x: hidden;
  transition: width 250ms ease, padding 250ms ease;
}

.sidebar.collapsed {
  width: var(--sidebar-collapsed-width);
  padding: var(--space-5) var(--space-2);
}

.sidebar-brand {
  padding: var(--space-2) var(--space-3);
  margin-bottom: var(--space-2);
  font-size: 15px;
  font-weight: 700;
  color: var(--text-primary);
  letter-spacing: -0.01em;
  white-space: nowrap;
  overflow: hidden;
  transition: opacity 250ms ease, max-height 250ms ease;
}

.sidebar.collapsed .sidebar-brand {
  opacity: 0;
  max-height: 0;
  margin-bottom: 0;
  padding: 0;
}

.sidebar-toggle {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 24px;
  height: 24px;
  background: var(--bg-surface-2);
  border: 1px solid var(--border-light);
  border-radius: var(--radius-sm);
  color: var(--text-muted);
  padding: 0;
  transition: color var(--transition), border-color var(--transition);
  flex-shrink: 0;
  align-self: flex-end;
  margin-bottom: var(--space-4);
}

.sidebar-toggle:hover {
  color: var(--text-primary);
  border-color: var(--accent);
}

.sidebar-toggle svg {
  width: 12px;
  height: 12px;
}

.sidebar-nav {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.sidebar-footer {
  padding-top: var(--space-4);
  border-top: 1px solid var(--border-light);
  display: flex;
  flex-direction: column;
  gap: var(--space-2);
}

.sidebar.collapsed .sidebar-footer {
  align-items: center;
}

.nav-item {
  display: flex;
  align-items: center;
  gap: var(--space-3);
  padding: var(--space-2) var(--space-3);
  border-radius: var(--radius-sm);
  color: var(--text-secondary);
  text-decoration: none;
  font-size: 13.5px;
  font-weight: 500;
  transition: background var(--transition), color var(--transition);
  white-space: nowrap;
  overflow: hidden;
  position: relative;
}

.sidebar.collapsed .nav-item {
  justify-content: center;
  padding: var(--space-2);
  gap: 0;
}

.nav-label {
  transition: opacity 200ms ease, width 200ms ease;
  opacity: 1;
  overflow: hidden;
}

.sidebar.collapsed .nav-label {
  opacity: 0;
  width: 0;
  display: none;
}

.sidebar.collapsed .nav-item[data-tooltip]::after {
  content: attr(data-tooltip);
  position: absolute;
  left: calc(100% + 8px);
  top: 50%;
  transform: translateY(-50%);
  background: var(--bg-surface);
  color: var(--text-primary);
  border: 1px solid var(--border-light);
  border-radius: var(--radius-sm);
  padding: 4px 10px;
  font-size: 12px;
  font-weight: 500;
  white-space: nowrap;
  pointer-events: none;
  opacity: 0;
  transition: opacity 150ms ease;
  z-index: 200;
  box-shadow: 0 4px 12px rgba(0,0,0,0.3);
}

.sidebar.collapsed .nav-item[data-tooltip]:hover::after {
  opacity: 1;
}

.nav-item:hover {
  background: var(--bg-hover);
  color: var(--text-primary);
}

.nav-item.router-link-active {
  background: var(--accent-subtle);
  color: var(--accent);
}

.nav-icon {
  width: 16px;
  height: 16px;
  flex-shrink: 0;
}

/* Card styles */
.card {
  background: var(--bg-surface);
  border: 1px solid var(--border-light);
  border-radius: var(--radius-lg);
  padding: var(--space-6);
  margin-bottom: var(--space-5);
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: var(--space-5);
  padding-bottom: var(--space-4);
  border-bottom: 1px solid var(--border-light);
}

.card-title {
  font-size: 14px;
  font-weight: 600;
  color: var(--text-primary);
  letter-spacing: -0.01em;
}

/* Table styles */
table { width: 100%; border-collapse: collapse; }
thead { background: var(--bg-surface-2); border-bottom: 1px solid var(--border-light); }
th {
  text-align: left;
  padding: 10px 14px;
  font-size: 11px;
  font-weight: 600;
  letter-spacing: 0.06em;
  text-transform: uppercase;
  color: var(--text-muted);
}
td {
  padding: 12px 14px;
  border-bottom: 1px solid var(--border);
  color: var(--text-secondary);
  font-size: 13px;
  vertical-align: middle;
}
tbody tr:last-child td { border-bottom: none; }
tbody tr:hover td { background: var(--bg-hover); }

/* Stats */
.stat-card {
  background: var(--bg-surface);
  border: 1px solid var(--border-light);
  border-radius: var(--radius-lg);
  padding: var(--space-5);
  transition: border-color var(--transition);
}
.stat-card:hover { border-color: var(--accent); }
.stat-label {
  color: var(--text-muted);
  font-size: 11px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  margin-bottom: var(--space-2);
}
.stat-value {
  font-size: 2rem;
  font-weight: 700;
  color: var(--text-primary);
  letter-spacing: -0.02em;
}
.stat-card.warning .stat-value { color: var(--status-amber); }
.stat-card.success .stat-value { color: var(--status-green); }
.stat-card.danger .stat-value  { color: var(--status-red); }
.stat-card.info .stat-value    { color: var(--status-blue); }
.stats-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(200px, 1fr)); gap: var(--space-4); margin-bottom: var(--space-6); }

/* Badges */
.badge {
  display: inline-flex;
  align-items: center;
  padding: 3px 8px;
  border-radius: 4px;
  font-size: 11px;
  font-weight: 600;
  letter-spacing: 0.02em;
}
.badge.success   { background: rgba(34,197,94,0.12);  color: #4ade80; border: 1px solid rgba(34,197,94,0.2); }
.badge.info      { background: rgba(59,130,246,0.12); color: #60a5fa; border: 1px solid rgba(59,130,246,0.2); }
.badge.warning   { background: rgba(245,158,11,0.12); color: #fbbf24; border: 1px solid rgba(245,158,11,0.2); }
.badge.danger    { background: rgba(239,68,68,0.12);  color: #f87171; border: 1px solid rgba(239,68,68,0.2); }
.badge.increasing { background: rgba(34,197,94,0.12); color: #4ade80; border: 1px solid rgba(34,197,94,0.2); }
.badge.decreasing { background: rgba(239,68,68,0.12); color: #f87171; border: 1px solid rgba(239,68,68,0.2); }
.badge.stable    { background: rgba(59,130,246,0.12); color: #60a5fa; border: 1px solid rgba(59,130,246,0.2); }
.badge.high      { background: rgba(239,68,68,0.12);  color: #f87171; border: 1px solid rgba(239,68,68,0.2); }
.badge.medium    { background: rgba(245,158,11,0.12); color: #fbbf24; border: 1px solid rgba(245,158,11,0.2); }
.badge.low       { background: rgba(59,130,246,0.12); color: #60a5fa; border: 1px solid rgba(59,130,246,0.2); }

/* Loading / Error */
.loading { text-align: center; padding: 3rem; color: var(--text-muted); font-size: 13px; }
.error { background: rgba(239,68,68,0.1); border: 1px solid rgba(239,68,68,0.2); color: #f87171; padding: var(--space-4); border-radius: var(--radius-md); margin: var(--space-4) 0; font-size: 13px; }

/* Table container */
.table-container { overflow-x: auto; }

/* Page header */
.page-header { display: flex; align-items: flex-start; justify-content: space-between; margin-bottom: var(--space-6); }
.page-header h2, .page-header h3 { font-size: 20px; font-weight: 600; color: var(--text-primary); letter-spacing: -0.01em; }
.page-header p { font-size: 13px; color: var(--text-muted); margin-top: 2px; }
</style>
