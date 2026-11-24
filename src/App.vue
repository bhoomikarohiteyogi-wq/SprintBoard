
<script setup>
import { ref, computed, watch, onMounted, onUnmounted } from 'vue'

const pages = ['Dashboard', 'My Tasks', 'Kanban Board', 'Team', 'Reports']
const statuses = ['To Do', 'In Progress', 'Testing', 'Done']
const priorities = ['High', 'Medium', 'Low']

const activePage = ref('Dashboard')
const search = ref('')
const showForm = ref(false)
const showNotifications = ref(false)
const editingId = ref(null)
const desktopAlerts = ref(false)
const clock = ref(new Date())

const people = [
  { name: 'Bhoomika', role: 'Project Manager', initials: 'BR', color: 'purple' },
  { name: 'Aishwarya', role: 'Developer', initials: 'AM', color: 'blue' },
  { name: 'Akshata', role: 'UI/UX Designer', initials: 'AT', color: 'pink' },
  { name: 'Sneha', role: 'Tester', initials: 'SS', color: 'green' }
]

function dateISO(offset = 0) {
  const date = new Date()
  date.setDate(date.getDate() + offset)
  return [
    date.getFullYear(),
    String(date.getMonth() + 1).padStart(2, '0'),
    String(date.getDate()).padStart(2, '0')
  ].join('-')
}

const initialTasks = [
  {
    id: 1,
    title: 'Finalize project requirements',
    description: 'Confirm scope, objectives and deliverables.',
    priority: 'High',
    status: 'Done',
    assignee: 'Bhoomika',
    dueDate: dateISO(-1)
  },
  {
    id: 2,
    title: 'Design the dashboard interface',
    description: 'Create a clean and responsive dashboard layout.',
    priority: 'High',
    status: 'In Progress',
    assignee: 'Akshata',
    dueDate: dateISO(1)
  },
  {
    id: 3,
    title: 'Develop task management module',
    description: 'Implement task creation, updates and deletion.',
    priority: 'Medium',
    status: 'To Do',
    assignee: 'Aishwarya',
    dueDate: dateISO(3)
  },
  {
    id: 4,
    title: 'Test deadline notifications',
    description: 'Verify reminders for approaching deadlines.',
    priority: 'High',
    status: 'Testing',
    assignee: 'Sneha',
    dueDate: dateISO(0)
  },
  {
    id: 5,
    title: 'Prepare final project presentation',
    description: 'Prepare slides and project demonstration.',
    priority: 'Medium',
    status: 'To Do',
    assignee: 'Bhoomika',
    dueDate: dateISO(5)
  }
]

function loadTasks() {
  try {
    const saved = localStorage.getItem('sprintboard-tasks')
    return saved ? JSON.parse(saved) : initialTasks
  } catch {
    return initialTasks
  }
}

const tasks = ref(loadTasks())

watch(tasks, value => {
  localStorage.setItem('sprintboard-tasks', JSON.stringify(value))
}, { deep: true })

const form = ref({
  title: '',
  description: '',
  priority: 'Medium',
  status: 'To Do',
  assignee: 'Bhoomika',
  dueDate: ''
})

const today = computed(() => dateISO())

const filteredTasks = computed(() => {
  const query = search.value.toLowerCase()
  return tasks.value.filter(task =>
    `${task.title} ${task.assignee} ${task.status}`
      .toLowerCase()
      .includes(query)
  )
})

const total = computed(() => tasks.value.length)

const completed = computed(() =>
  tasks.value.filter(task => task.status === 'Done').length
)

const progress = computed(() =>
  total.value ? Math.round(completed.value / total.value * 100) : 0
)

const inProgress = computed(() =>
  tasks.value.filter(task => task.status === 'In Progress').length
)

const overdue = computed(() =>
  tasks.value.filter(task =>
    task.status !== 'Done' &&
    task.dueDate &&
    task.dueDate < today.value
  )
)

const dueToday = computed(() =>
  tasks.value.filter(task =>
    task.status !== 'Done' &&
    task.dueDate === today.value
  )
)

const upcoming = computed(() =>
  tasks.value.filter(task =>
    task.status !== 'Done' &&
    task.dueDate > today.value &&
    task.dueDate <= dateISO(2)
  )
)

const alerts = computed(() => [
  ...overdue.value.map(task => ({
    ...task,
    kind: 'overdue',
    message: 'Deadline has passed'
  })),
  ...dueToday.value.map(task => ({
    ...task,
    kind: 'today',
    message: 'Due today'
  })),
  ...upcoming.value.map(task => ({
    ...task,
    kind: 'soon',
    message: 'Due within 48 hours'
  }))
])

const dateLabel = computed(() =>
  clock.value.toLocaleDateString('en-IN', {
    weekday: 'long',
    day: 'numeric',
    month: 'long',
    year: 'numeric'
  })
)

function formatDate(value) {
  if (!value) return 'Not set'
  return new Date(value + 'T00:00:00').toLocaleDateString('en-IN', {
    day: 'numeric',
    month: 'short',
    year: 'numeric'
  })
}

function newTask() {
  editingId.value = null
  form.value = {
    title: '',
    description: '',
    priority: 'Medium',
    status: 'To Do',
    assignee: 'Bhoomika',
    dueDate: ''
  }
  showForm.value = true
}

function addSubmissionTask() {
  editingId.value = null
  form.value = {
    title: 'Final Project Submission',
    description: 'Submit the completed SprintBoard project.',
    priority: 'High',
    status: 'To Do',
    assignee: 'Bhoomika',
    dueDate: ''
  }
  showForm.value = true
}

function editTask(task) {
  editingId.value = task.id
  form.value = { ...task }
  showForm.value = true
}

function saveTask() {
  if (!form.value.title.trim()) {
    alert('Please enter a task title.')
    return
  }

  if (editingId.value !== null) {
    const index = tasks.value.findIndex(
      task => task.id === editingId.value
    )

    if (index !== -1) {
      tasks.value[index] = {
        ...form.value,
        id: editingId.value,
        title: form.value.title.trim()
      }
    }
  } else {
    tasks.value.unshift({
      ...form.value,
      id: Date.now(),
      title: form.value.title.trim()
    })
  }

  showForm.value = false
}

function removeTask(id) {
  if (confirm('Are you sure you want to delete this task?')) {
    tasks.value = tasks.value.filter(task => task.id !== id)
  }
}

function dragStart(event, task) {
  event.dataTransfer.setData('text/plain', String(task.id))
}

function dropTask(event, status) {
  const id = Number(event.dataTransfer.getData('text/plain'))
  const task = tasks.value.find(item => item.id === id)

  if (task) task.status = status
}

function enableNotifications() {
  if (!('Notification' in window)) {
    alert('Your browser does not support desktop notifications.')
    return
  }

  Notification.requestPermission().then(permission => {
    desktopAlerts.value = permission === 'granted'

    if (desktopAlerts.value) {
      new Notification('SprintBoard alerts enabled', {
        body: 'Deadline reminders are enabled while this website is open.'
      })
      checkDesktopNotifications()
    } else {
      alert('Please allow notifications in your browser settings.')
    }
  })
}

function checkDesktopNotifications() {
  if (
    !desktopAlerts.value ||
    !('Notification' in window) ||
    Notification.permission !== 'granted'
  ) return

  alerts.value.forEach(task => {
    const key = `sprintboard-${task.id}-${task.kind}-${task.dueDate}`

    if (localStorage.getItem(key)) return

    new Notification('SprintBoard deadline reminder', {
      body: `${task.title} — ${task.message}`,
      tag: key
    })

    localStorage.setItem(key, 'sent')
  })
}

function exportTasks() {
  const blob = new Blob(
    [JSON.stringify(tasks.value, null, 2)],
    { type: 'application/json' }
  )

  const url = URL.createObjectURL(blob)
  const link = document.createElement('a')

  link.href = url
  link.download = 'sprintboard-tasks.json'
  link.click()

  URL.revokeObjectURL(url)
}

let reminderTimer
let clockTimer

onMounted(() => {
  desktopAlerts.value =
    'Notification' in window &&
    Notification.permission === 'granted'

  checkDesktopNotifications()

  reminderTimer = setInterval(() => {
    checkDesktopNotifications()
  }, 60000)

  clockTimer = setInterval(() => {
    clock.value = new Date()
  }, 60000)
})

onUnmounted(() => {
  clearInterval(reminderTimer)
  clearInterval(clockTimer)
})
</script>

<template>
  <div class="shell">
    <aside class="sidebar">
      <div class="brand">
        <span class="brand-icon">S</span>
        <span>
          <b>SprintBoard</b>
          <small>PROJECT WORKSPACE</small>
        </span>
      </div>

      <div class="workspace">
        <span class="workspace-mark">SB</span>
        <span>
          <b>Student Project</b>
          <small>Free workspace</small>
        </span>
        <span class="chevron">⌄</span>
      </div>

      <p class="nav-label">WORKSPACE</p>

      <nav>
        <button
          v-for="(page, index) in pages"
          :key="page"
          :class="{ active: activePage === page }"
          @click="activePage = page"
        >
          <span class="nav-symbol">
            {{ ['◫', '☑', '▦', '♧', '▤'][index] }}
          </span>
          {{ page }}
          <span v-if="page === 'My Tasks'" class="nav-number">
            {{ tasks.filter(task => task.status !== 'Done').length }}
          </span>
        </button>
      </nav>

      <div class="project-area">
        <p class="nav-label">YOUR PROJECT</p>
        <div class="project-name">
          <i></i>
          SprintBoard
          <span>•••</span>
        </div>
      </div>

      <div class="profile">
        <div class="avatar purple">BR</div>
        <div>
          <b>Bhoomika Rohite</b>
          <small>Project Manager</small>
        </div>
        <span>•••</span>
      </div>
    </aside>

    <main class="main">
      <header class="topbar">
        <div class="crumb">
          Workspace <span>/</span> {{ activePage }}
        </div>

        <div class="top-right">
          <label class="search">
            <span>⌕</span>
            <input v-model="search" placeholder="Search tasks..." />
          </label>

          <div class="notification-wrap">
            <button
              class="bell"
              @click="showNotifications = !showNotifications"
              aria-label="Open deadline notifications"
            >
              ♧
              <i v-if="alerts.length"></i>
              <span v-if="alerts.length" class="bell-count">
                {{ alerts.length }}
              </span>
            </button>

            <div v-if="showNotifications" class="notification-menu">
              <div class="notification-head">
                <div>
                  <b>Deadline Notifications</b>
                  <small>{{ alerts.length }} reminders need attention</small>
                </div>
                <button @click="showNotifications = false">×</button>
              </div>

              <button class="enable" @click="enableNotifications">
                {{ desktopAlerts ? '✓ Desktop alerts enabled' : 'Enable desktop notifications' }}
              </button>

              <div v-if="alerts.length">
                <div
                  v-for="alert in alerts"
                  :key="alert.id + alert.kind"
                  class="alert-row"
                >
                  <span class="alert-icon" :class="alert.kind">!</span>
                  <div>
                    <b>{{ alert.title }}</b>
                    <p>
                      {{ alert.message }} · {{ formatDate(alert.dueDate) }}
                    </p>
                  </div>
                </div>
              </div>

              <p v-else class="empty">
                You're all caught up. No deadline alerts.
              </p>

              <button class="submission-shortcut" @click="addSubmissionTask">
                + Add project submission deadline
              </button>
            </div>
          </div>

          <div class="avatar top-avatar">BR</div>
        </div>
      </header>

      <section class="content">
        <template v-if="activePage === 'Dashboard'">
          <div class="welcome">
            <div>
              <div class="eyebrow">PROJECT OVERVIEW</div>
              <h1>Welcome back, Bhoomika <span>✦</span></h1>
              <p>Here's what's happening with your project today.</p>
            </div>
            <div class="date-chip">◷ {{ dateLabel }}</div>
          </div>

          <div class="hero">
            <div>
              <span class="hero-label">YOUR WORKSPACE AT A GLANCE</span>
              <h2>Make progress.<br /><em>One task at a time.</em></h2>
              <p>Plan clearly, collaborate smoothly, and deliver on time.</p>
              <button @click="activePage = 'Kanban Board'">
                View project board <b>→</b>
              </button>
            </div>

            <div class="hero-graphic">
              <div class="orb orb-a"></div>
              <div class="orb orb-b"></div>
              <div class="floating-card">
                <div class="float-check">✓</div>
                <b>Project progress</b>
                <strong>{{ progress }}%</strong>
                <div class="float-track">
                  <i :style="{ width: progress + '%' }"></i>
                </div>
                <small>You're making great progress</small>
              </div>
              <span class="sparkle">✦</span>
            </div>
          </div>

          <div class="stats">
            <article class="stat">
              <div><span class="stat-icon blue">▦</span><small>ALL TASKS</small></div>
              <strong>{{ total }}</strong>
              <p>Total tasks</p>
              <label>Across your workspace</label>
            </article>

            <article class="stat">
              <div><span class="stat-icon green">✓</span><small>COMPLETED</small></div>
              <strong>{{ completed }}</strong>
              <p>Completed tasks</p>
              <label>{{ progress }}% of total tasks</label>
            </article>

            <article class="stat">
              <div><span class="stat-icon orange">◷</span><small>IN PROGRESS</small></div>
              <strong>{{ inProgress }}</strong>
              <p>In progress</p>
              <label>Currently being worked on</label>
            </article>

            <article class="stat">
              <div><span class="stat-icon red">⚑</span><small>NEEDS ATTENTION</small></div>
              <strong>{{ overdue.length + dueToday.length }}</strong>
              <p>Deadline alerts</p>
              <label>Overdue or due today</label>
            </article>
          </div>

          <div class="two-col">
            <article class="panel">
              <div class="panel-head">
                <div>
                  <h3>Project progress</h3>
                  <p>Track your team's overall completion</p>
                </div>
                <span class="live">LIVE</span>
              </div>

              <div class="progress-content">
                <div class="ring" :style="{ '--value': progress + '%' }">
                  <div>
                    <b>{{ progress }}%</b>
                    <small>Complete</small>
                  </div>
                </div>

                <div class="legend">
                  <div><i class="dot d-green"></i>Completed <b>{{ completed }}</b></div>
                  <div><i class="dot d-blue"></i>In progress <b>{{ inProgress }}</b></div>
                  <div>
                    <i class="dot d-gray"></i>
                    To do / Testing
                    <b>{{ tasks.filter(t => t.status === 'To Do' || t.status === 'Testing').length }}</b>
                  </div>
                </div>
              </div>
            </article>

            <article class="panel">
              <div class="panel-head">
                <div>
                  <h3>Team members</h3>
                  <p>People making it happen</p>
                </div>
                <button class="link" @click="activePage = 'Team'">View all →</button>
              </div>

              <div v-for="member in people" :key="member.name" class="member-row">
                <div class="avatar" :class="member.color">{{ member.initials }}</div>
                <div class="member-text">
                  <b>{{ member.name }}</b>
                  <small>{{ member.role }}</small>
                </div>
                <i class="online"></i>
              </div>
            </article>
          </div>

          <article class="panel task-panel">
            <div class="panel-head">
              <div>
                <h3>Recent tasks</h3>
                <p>Keep an eye on your latest project activities</p>
              </div>
              <button class="link" @click="activePage = 'My Tasks'">
                View all tasks →
              </button>
            </div>

            <div class="table-scroll">
              <table>
                <thead>
                  <tr>
                    <th>TASK NAME</th>
                    <th>ASSIGNEE</th>
                    <th>PRIORITY</th>
                    <th>STATUS</th>
                    <th>DEADLINE</th>
                    <th></th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-for="task in filteredTasks.slice(0, 5)" :key="task.id">
                    <td>
                      <b>{{ task.title }}</b>
                      <small class="sub">{{ task.description }}</small>
                    </td>
                    <td>{{ task.assignee }}</td>
                    <td><span class="priority" :class="task.priority.toLowerCase()">{{ task.priority }}</span></td>
                    <td><span class="status" :class="task.status.toLowerCase().replace(' ', '-')">{{ task.status }}</span></td>
                    <td>{{ formatDate(task.dueDate) }}</td>
                    <td><button class="edit" @click="editTask(task)">Edit</button></td>
                  </tr>
                  <tr v-if="!filteredTasks.length">
                    <td colspan="6" class="empty">No tasks found.</td>
                  </tr>
                </tbody>
              </table>
            </div>
          </article>
        </template>

        <template v-else-if="activePage === 'My Tasks'">
          <div class="page-title">
            <div>
              <div class="eyebrow">WORK MANAGEMENT</div>
              <h1>My Tasks</h1>
              <p>Manage and keep track of every task in your project.</p>
            </div>
            <button class="primary" @click="newTask">＋ Add new task</button>
          </div>

          <article class="panel task-panel">
            <div class="panel-head">
              <div>
                <h3>All project tasks</h3>
                <p>{{ filteredTasks.length }} tasks found</p>
              </div>
              <button class="secondary" @click="exportTasks">↓ Export tasks</button>
            </div>

            <div class="table-scroll">
              <table>
                <thead>
                  <tr>
                    <th>TASK NAME</th>
                    <th>ASSIGNEE</th>
                    <th>PRIORITY</th>
                    <th>STATUS</th>
                    <th>DEADLINE</th>
                    <th>ACTIONS</th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-for="task in filteredTasks" :key="task.id">
                    <td>
                      <b>{{ task.title }}</b>
                      <small class="sub">{{ task.description }}</small>
                    </td>
                    <td>{{ task.assignee }}</td>
                    <td><span class="priority" :class="task.priority.toLowerCase()">{{ task.priority }}</span></td>
                    <td>
                      <select class="status-select" v-model="task.status">
                        <option v-for="status in statuses" :key="status">{{ status }}</option>
                      </select>
                    </td>
                    <td>{{ formatDate(task.dueDate) }}</td>
                    <td>
                      <button class="edit" @click="editTask(task)">Edit</button>
                      <button class="delete" @click="removeTask(task.id)">×</button>
                    </td>
                  </tr>
                </tbody>
              </table>
            </div>
          </article>
        </template>

        <template v-else-if="activePage === 'Kanban Board'">
          <div class="page-title">
            <div>
              <div class="eyebrow">VISUAL WORKFLOW</div>
              <h1>Kanban Board</h1>
              <p>Drag tasks between columns to update their status.</p>
            </div>
            <button class="primary" @click="newTask">＋ Add new task</button>
          </div>

          <div class="kanban">
            <section
              v-for="(status, index) in statuses"
              :key="status"
              class="kanban-col"
              @dragover.prevent
              @drop="dropTask($event, status)"
            >
              <div class="col-head">
                <i :class="'c' + index"></i>
                <b>{{ status }}</b>
                <span>{{ filteredTasks.filter(t => t.status === status).length }}</span>
              </div>

              <article
                v-for="task in filteredTasks.filter(t => t.status === status)"
                :key="task.id"
                class="kanban-card"
                draggable="true"
                @dragstart="dragStart($event, task)"
              >
                <div class="card-meta">
                  <span class="priority" :class="task.priority.toLowerCase()">{{ task.priority }}</span>
                  <button @click="editTask(task)">•••</button>
                </div>
                <h3>{{ task.title }}</h3>
                <p>{{ task.description }}</p>
                <div class="card-foot">
                  ◷ {{ formatDate(task.dueDate) }}
                  <span>{{ task.assignee.slice(0, 1) }}</span>
                </div>
              </article>

              <button class="add-card" @click="newTask">＋ Add a task</button>
            </section>
          </div>
        </template>

        <template v-else-if="activePage === 'Team'">
          <div class="page-title">
            <div>
              <div class="eyebrow">COLLABORATION</div>
              <h1>Your Team</h1>
              <p>Meet the people behind the SprintBoard project.</p>
            </div>
          </div>

          <div class="team-grid">
            <article v-for="member in people" :key="member.name" class="panel team-card">
              <div class="avatar big" :class="member.color">{{ member.initials }}</div>
              <h3>{{ member.name }}</h3>
              <p>{{ member.role }}</p>
              <span>● Active member</span>
            </article>
          </div>
        </template>

        <template v-else>
          <div class="page-title">
            <div>
              <div class="eyebrow">PROJECT INSIGHTS</div>
              <h1>Reports</h1>
              <p>A quick summary of your team's project performance.</p>
            </div>
            <button class="secondary" @click="exportTasks">↓ Export tasks</button>
          </div>

          <div class="stats report-stats">
            <article class="stat"><strong>{{ total }}</strong><p>Total tasks</p></article>
            <article class="stat"><strong>{{ completed }}</strong><p>Completed</p></article>
            <article class="stat"><strong>{{ progress }}%</strong><p>Completion rate</p></article>
            <article class="stat"><strong>{{ overdue.length }}</strong><p>Overdue tasks</p></article>
          </div>

          <article class="panel report">
            <div class="panel-head">
              <div>
                <h3>Task status breakdown</h3>
                <p>Distribution across workflow stages</p>
              </div>
            </div>

            <div v-for="status in statuses" :key="status" class="report-line">
              <div>
                {{ status }}
                <b>{{ tasks.filter(t => t.status === status).length }}</b>
              </div>
              <div class="bar">
                <i :style="{ width: (total ? tasks.filter(t => t.status === status).length / total * 100 : 0) + '%' }"></i>
              </div>
            </div>

            <button class="primary" @click="exportTasks">Export project data ↓</button>
          </article>
        </template>
      </section>

      <footer>
        © 2026 SprintBoard · Agile Project Management System
        <span>Built for better teamwork ✦</span>
      </footer>
    </main>

    <div v-if="showForm" class="modal-bg" @click.self="showForm = false">
      <form class="modal" @submit.prevent="saveTask">
        <div class="modal-head">
          <div>
            <div class="eyebrow">TASK MANAGEMENT</div>
            <h2>{{ editingId !== null ? 'Edit task' : 'Create a new task' }}</h2>
          </div>
          <button type="button" class="close" @click="showForm = false">×</button>
        </div>

        <label>
          Task title
          <input v-model="form.title" required placeholder="e.g. Prepare project presentation" />
        </label>

        <label>
          Description
          <textarea v-model="form.description" rows="3" placeholder="Add a short description..."></textarea>
        </label>

        <div class="form-grid">
          <label>
            Priority
            <select v-model="form.priority">
              <option v-for="priority in priorities" :key="priority">{{ priority }}</option>
            </select>
          </label>

          <label>
            Status
            <select v-model="form.status">
              <option v-for="status in statuses" :key="status">{{ status }}</option>
            </select>
          </label>

          <label>
            Assignee
            <select v-model="form.assignee">
              <option v-for="member in people" :key="member.name">{{ member.name }}</option>
            </select>
          </label>

          <label>
            Project deadline
            <input v-model="form.dueDate" type="date" />
          </label>
        </div>

        <div class="modal-actions">
          <button type="button" class="secondary" @click="showForm = false">Cancel</button>
          <button class="primary">{{ editingId !== null ? 'Save changes' : 'Create task' }}</button>
        </div>
      </form>
    </div>
  </div>
</template>
