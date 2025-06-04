<template>
  <!-- Offline Alert Banner -->
  <transition name="fade-slide" appear>
    <div
      v-if="!isOnline"
      class="fixed top-2 left-1/2 transform -translate-x-1/2 bg-yellow-100 text-yellow-800 px-4 py-2 rounded shadow text-sm z-50 min-w-[320px]"
    >
      You are offline. Changes will be synced when you're back online.
    </div>
  </transition>

  <!-- PWA Install Prompt -->
  <PwaInstallPrompt
    :showInstallPrompt="showInstallPrompt"
    :isOnline="isOnline"
    @install="promptInstall"
  />

  <!-- Main UI -->
  <div class="min-h-screen bg-gray-100 flex items-center justify-center">
    <div class="bg-white p-8 rounded shadow-md w-full max-w-md">
      <h1 class="text-2xl font-bold mb-4 text-center">Todo App</h1>

      <!-- Add Todo Form -->
      <form @submit.prevent="addTodo" class="flex mb-4">
        <input
          v-model="newTodo"
          type="text"
          placeholder="Add a new todo"
          class="flex-1 p-2 border rounded-l focus:outline-none"
        />
        <button type="submit" class="bg-blue-500 text-white px-4 rounded-r hover:bg-blue-600">
          Add
        </button>
      </form>

      <!-- Todo List -->
      <ul>
        <li
          v-for="todo in todos"
          :key="todo.id"
          class="flex items-center justify-between p-2 border-b last:border-b-0"
        >
          <div class="flex items-center gap-2">
            <!-- Pending Indicator -->
            <span
              v-if="todo.pendingAction"
              class="w-2 h-2 rounded-full bg-yellow-500 inline-block"
              title="Pending offline action"
            ></span>

            <!-- Completion Toggle -->
            <input
              type="checkbox"
              v-model="todo.completed"
              class="mr-2"
              @change="updateTodo(todo)"
              :disabled="todo.pendingAction === 'delete'"
            />

            <!-- Todo Text -->
            <span
              :class="{
                'line-through text-gray-400': todo.completed,
                'opacity-60': todo.pendingAction,
              }"
            >
              {{ todo.text }}
            </span>
          </div>

          <!-- Delete Button -->
          <button
            @click="deleteTodo(todo.id)"
            class="text-red-500 hover:text-red-700 disabled:opacity-50"
            title="Delete"
            :disabled="todo.pendingAction === 'delete'"
          >
            &times;
          </button>
        </li>
      </ul>

      <!-- Empty State -->
      <div v-if="todos.length === 0" class="text-center text-gray-400 mt-4">No todos yet!</div>
    </div>

    <!-- Service Worker Update UI -->
    <Update />
  </div>
</template>

<script lang="ts" setup>
import { ref, onMounted, watch } from 'vue'
import Update from './components/Update.vue'
import PwaInstallPrompt from './components/PwaInstallPrompt.vue'

// --- Types ---
interface Todo {
  id: number
  text: string
  completed: boolean
  pendingAction?: 'add' | 'update' | 'delete' | null
}

interface Queue {
  url: string
  method: 'POST' | 'PUT' | 'DELETE'
  body: string | null
}

// --- State ---
const todos = ref<Todo[]>([])
const newTodo = ref('')
const queues = ref<Queue[]>([])
const isOnline = ref(navigator.onLine)
const userId = 5 // dummy userId for API

// --- Constants ---
const API_BASE = 'https://dummyjson.com/todos'
const QUEUE_KEY = 'todo-queues'

// --- PWA Install Prompt ---
const deferredPrompt = ref<Event | null>(null)
const showInstallPrompt = ref(false)

// --- Restore queue and setup listeners on mount ---
onMounted(() => {
  queues.value = JSON.parse(localStorage.getItem(QUEUE_KEY) || '[]')
  fetchTodos()
  processQueue()

  window.addEventListener('online', () => (isOnline.value = true))
  window.addEventListener('offline', () => (isOnline.value = false))

  window.addEventListener('beforeinstallprompt', (e: Event) => {
    e.preventDefault()
    deferredPrompt.value = e
    showInstallPrompt.value = true
  })
})

// Watch online status and sync when online
watch(isOnline, (online) => {
  if (online) processQueue()
})

// Persist queue to localStorage (debounced)
let saveTimeout: number | null = null
watch(
  queues,
  (val) => {
    if (saveTimeout) clearTimeout(saveTimeout)
    saveTimeout = setTimeout(() => {
      localStorage.setItem(QUEUE_KEY, JSON.stringify(val))
    }, 300)
  },
  { deep: true },
)

// --- API Logic ---

const fetchTodos = async () => {
  try {
    const res = await fetch(`${API_BASE}?limit=5`)
    const data = await res.json()
    todos.value = data.todos.map((t: any) => ({
      id: t.id,
      text: t.todo,
      completed: t.completed,
    }))
  } catch (err) {
    console.error('Failed to fetch todos:', err)
  }
}

const apiRequest = async (queue: Queue) => {
  const headers = queue.body ? { 'Content-Type': 'application/json' } : undefined
  const res = await fetch(queue.url, {
    method: queue.method,
    headers,
    body: queue.body,
  })
  return res.json()
}

// --- Queue Handling ---

const processQueue = async () => {
  const pending = [...queues.value]
  const stillPending: Queue[] = []

  for (const q of pending) {
    try {
      const body = q.body ? safeParse(q.body) : {}
      const data = await apiRequest(q)

      switch (q.method) {
        case 'POST': {
          const idx = todos.value.findIndex((t) => t.id === body.tempId)
          const newItem = {
            id: data.id,
            text: data.todo,
            completed: data.completed,
            pendingAction: null,
          }
          idx !== -1 ? todos.value.splice(idx, 1, newItem) : todos.value.push(newItem)
          break
        }
        case 'PUT': {
          const idx = todos.value.findIndex((t) => t.id === body.tempId || t.id === data.id)
          if (idx !== -1) {
            todos.value[idx].completed = data.completed
            todos.value[idx].pendingAction = null
          }
          break
        }
        case 'DELETE': {
          todos.value = todos.value.filter((t) => t.id !== body.tempId && t.id !== data.id)
          break
        }
      }
    } catch (err) {
      console.error('Queue item failed:', err)
      stillPending.push(q)
    }
  }

  queues.value = stillPending
}

// --- Add Todo ---

const addTodo = async () => {
  const text = newTodo.value.trim()
  if (!text) return

  newTodo.value = ''
  const tempId = Date.now() * -1
  const body = { todo: text, completed: false, userId }
  const queue: Queue = {
    url: `${API_BASE}/add`,
    method: 'POST',
    body: JSON.stringify({ ...body, tempId }),
  }

  if (isOnline.value) {
    try {
      const data = await apiRequest(queue)
      todos.value.push({
        id: data.id,
        text: data.todo,
        completed: data.completed,
        pendingAction: null,
      })
      return
    } catch {
      console.warn('API failed, queuing instead')
    }
  }

  addPendingTodo({ id: tempId, text, completed: false })
  queues.value.push(queue)
}

// --- Delete Todo ---

const deleteTodo = async (id: number) => {
  const todo = todos.value.find((t) => t.id === id)
  if (!todo) return

  if (todo.pendingAction === 'add') {
    todos.value = todos.value.filter((t) => t.id !== id)
    queues.value = queues.value.filter((q) => safeParse(q.body)?.tempId !== id)
    return
  }

  const queue: Queue = {
    url: `${API_BASE}/${id}`,
    method: 'DELETE',
    body: JSON.stringify({ tempId: id }),
  }

  if (isOnline.value) {
    try {
      await apiRequest(queue)
      todos.value = todos.value.filter((t) => t.id !== id)
    } catch {
      fallbackQueueDelete(id, queue)
    }
  } else {
    fallbackQueueDelete(id, queue)
  }
}

const fallbackQueueDelete = (id: number, queue: Queue) => {
  markPendingAction(id, 'delete')
  queues.value.push(queue)
}

// --- Update Todo ---

const updateTodo = async (todo: Todo) => {
  const body = JSON.stringify({ completed: todo.completed })
  const queue: Queue = { url: `${API_BASE}/${todo.id}`, method: 'PUT', body }

  if (isOnline.value) {
    try {
      const data = await apiRequest(queue)
      const t = todos.value.find((t) => t.id === todo.id)
      if (t) {
        t.completed = data.completed
        t.pendingAction = null
      }
    } catch {
      fallbackQueueUpdate(todo.id, queue)
    }
  } else {
    fallbackQueueUpdate(todo.id, queue)
  }
}

const fallbackQueueUpdate = (id: number, queue: Queue) => {
  markPendingAction(id, 'update')
  queues.value.push(queue)
}

// --- Helpers ---

const markPendingAction = (id: number, action: 'add' | 'update' | 'delete') => {
  const todo = todos.value.find((t) => t.id === id)
  if (todo) todo.pendingAction = action
}

const addPendingTodo = (todo: Todo) => {
  todos.value.push({ ...todo, pendingAction: 'add' })
}

const safeParse = (json: string | null) => {
  try {
    return json ? JSON.parse(json) : {}
  } catch {
    return {}
  }
}

const promptInstall = async () => {
  if (deferredPrompt.value) {
    // @ts-ignore
    deferredPrompt.value.prompt()
    // @ts-ignore
    const { outcome } = await deferredPrompt.value.userChoice
    if (outcome === 'accepted') {
      showInstallPrompt.value = false
      deferredPrompt.value = null
    }
  }
}
</script>

<style>
body {
  margin: 0;
  font-family: system-ui, sans-serif;
}

/* Fade and slide-down animation for banners */
.fade-slide-enter-active,
.fade-slide-leave-active {
  transition:
    opacity 0.3s,
    transform 0.3s;
}
.fade-slide-enter-from,
.fade-slide-leave-to {
  opacity: 0;
  transform: translateY(-16px);
}
.fade-slide-enter-to,
.fade-slide-leave-from {
  opacity: 1;
  transform: translateY(0);
}
</style>
