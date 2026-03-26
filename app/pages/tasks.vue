<script setup lang="ts">
definePageMeta({ middleware: 'auth' })

const toast = useToast()
const { openConfirm } = useConfirm()
const { data: tasks, refresh } = await useFetch('/api/tasks')

const showModal = ref(false)
const editingTask = ref<any>(null)
const form = reactive({ title: '', description: '', priority: 'medium', dueDate: '' })
const loading = ref(false)

const draggingTask = ref<any>(null)
const dragOverColumn = ref<string | null>(null)

const columns = [
  { key: 'todo', label: 'To Do', icon: 'i-lucide-circle', color: 'text-neutral-400' },
  { key: 'in_progress', label: 'In Progress', icon: 'i-lucide-circle-dot', color: 'text-warning' },
  { key: 'done', label: 'Done', icon: 'i-lucide-circle-check-big', color: 'text-success' },
]

const tasksByStatus = computed(() => ({
  todo: (tasks.value ?? []).filter(t => t.status === 'todo'),
  in_progress: (tasks.value ?? []).filter(t => t.status === 'in_progress'),
  done: (tasks.value ?? []).filter(t => t.status === 'done'),
}))

function openCreate() {
  editingTask.value = null
  Object.assign(form, { title: '', description: '', priority: 'medium', dueDate: '' })
  showModal.value = true
}

function openEdit(task: any) {
  editingTask.value = task
  Object.assign(form, {
    title: task.title,
    description: task.description ?? '',
    priority: task.priority,
    dueDate: task.dueDate ?? '',
  })
  showModal.value = true
}

function confirmDelete(task: any) {
  openConfirm(
    'Delete task?',
    `'${task.title}' will be permanently deleted.`,
    async () => {
      await $fetch(`/api/tasks/${task.id}`, { method: 'DELETE' })
      toast.add({ title: 'Task deleted', color: 'neutral' })
      await refresh()
    },
  )
}

async function save() {
  loading.value = true
  try {
    if (editingTask.value) {
      await $fetch(`/api/tasks/${editingTask.value.id}`, { method: 'PATCH', body: form })
      toast.add({ title: 'Task updated', color: 'success' })
    }
    else {
      await $fetch('/api/tasks', { method: 'POST', body: form })
      toast.add({ title: 'Task created', color: 'success' })
    }
    showModal.value = false
    await refresh()
  }
  catch (err: any) {
    toast.add({ title: 'Error', description: err.data?.message, color: 'error' })
  }
  finally {
    loading.value = false
  }
}

function onDragStart(task: any) {
  draggingTask.value = task
}

function onDragEnd() {
  draggingTask.value = null
  dragOverColumn.value = null
}

async function onDrop(status: string) {
  const task = draggingTask.value
  draggingTask.value = null
  dragOverColumn.value = null

  if (!task || task.status === status) return

  // Optimistic update
  const local = tasks.value?.find(t => t.id === task.id)
  if (local) local.status = status

  try {
    await $fetch(`/api/tasks/${task.id}`, { method: 'PATCH', body: { status } })
    await refresh()
  }
  catch {
    toast.add({ title: 'Failed to move task', color: 'error' })
    await refresh()
  }
}

const priorityColor = (p: string) => p === 'high' ? 'error' : p === 'medium' ? 'warning' : 'neutral'
const priorityBorder = (p: string) => p === 'high'
  ? 'border-l-4 border-l-error'
  : p === 'medium'
    ? 'border-l-4 border-l-warning'
    : 'border-l-4 border-l-neutral-200 dark:border-l-neutral-700'
</script>

<template>
  <div class="h-screen flex flex-col overflow-hidden">
    <!-- Header -->
    <div class="px-4 lg:px-8 py-4 border-b border-gray-200 flex items-center justify-between flex-shrink-0">
      <div>
        <h1 class="text-2xl font-bold">Tasks</h1>
        <p class="text-sm text-muted-foreground mt-0.5">
          {{ tasksByStatus.done.length }} of {{ tasks?.length ?? 0 }} completed
        </p>
      </div>
      <UButton icon="i-lucide-plus" @click="openCreate">New Task</UButton>
    </div>

    <!-- Kanban board -->
    <div class="flex-1 overflow-x-auto overflow-y-hidden p-4 lg:p-6">
      <div class="flex gap-4 h-full min-w-[680px]">
        <div
          v-for="col in columns"
          :key="col.key"
          class="flex-1 flex flex-col rounded-xl transition-all duration-150"
          :class="dragOverColumn === col.key
            ? 'bg-primary/5 ring-2 ring-primary/25'
            : 'bg-muted/50'"
          @dragover.prevent="dragOverColumn = col.key"
          @dragleave="dragOverColumn = null"
          @drop.prevent="onDrop(col.key)"
        >
          <!-- Column header -->
          <div class="px-3 pt-3 pb-2 flex items-center gap-2 flex-shrink-0">
            <UIcon :name="col.icon" class="text-base" :class="col.color" />
            <span class="font-semibold text-sm">{{ col.label }}</span>
            <span class="ml-auto text-xs font-medium bg-background rounded-full px-2 py-0.5 border border-gray-200">
              {{ tasksByStatus[col.key as keyof typeof tasksByStatus].length }}
            </span>
          </div>

          <!-- Task cards -->
          <div class="flex-1 overflow-y-auto px-2 pb-2 space-y-2">
            <div
              v-for="task in tasksByStatus[col.key as keyof typeof tasksByStatus]"
              :key="task.id"
              draggable="true"
              class="bg-card rounded-lg p-3 shadow-sm select-none cursor-grab active:cursor-grabbing transition-all"
              :class="[
                priorityBorder(task.priority),
                draggingTask?.id === task.id ? 'opacity-40 scale-[0.97]' : 'hover:shadow-md',
              ]"
              @dragstart="onDragStart(task)"
              @dragend="onDragEnd"
            >
              <p
                class="text-sm font-medium leading-snug"
                :class="task.status === 'done' ? 'line-through text-muted-foreground' : ''"
              >
                {{ task.title }}
              </p>
              <p v-if="task.description" class="text-xs text-muted-foreground mt-1 line-clamp-2 leading-relaxed">
                {{ task.description }}
              </p>
              <div class="flex items-center gap-1.5 mt-2">
                <UBadge :color="priorityColor(task.priority)" variant="soft" size="xs">
                  {{ task.priority }}
                </UBadge>
                <span v-if="task.dueDate" class="text-xs text-muted-foreground flex items-center gap-1">
                  <UIcon name="i-lucide-calendar" class="text-xs" />
                  {{ task.dueDate }}
                </span>
                <div class="ml-auto flex gap-0.5">
                  <UButton icon="i-lucide-pencil" variant="ghost" size="xs" color="neutral" @click.stop="openEdit(task)" />
                  <UButton icon="i-lucide-trash-2" variant="ghost" size="xs" color="error" @click.stop="confirmDelete(task)" />
                </div>
              </div>
            </div>

            <div
              v-if="!tasksByStatus[col.key as keyof typeof tasksByStatus].length"
              class="py-10 text-center text-muted-foreground"
              :class="dragOverColumn === col.key ? 'opacity-60' : 'opacity-40'"
            >
              <UIcon name="i-lucide-inbox" class="text-3xl mb-2" />
              <p class="text-xs">Drop tasks here</p>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Create / Edit Modal -->
    <UModal v-model:open="showModal" :title="editingTask ? 'Edit Task' : 'New Task'">
      <template #body>
        <UForm :state="form" class="space-y-4" @submit="save">
          <UFormField label="Title" name="title" required>
            <UInput v-model="form.title" placeholder="What needs to be done?" class="w-full" autofocus />
          </UFormField>
          <UFormField label="Description" name="description">
            <UTextarea v-model="form.description" placeholder="Add details (optional)" class="w-full" :rows="3" />
          </UFormField>
          <div class="grid grid-cols-2 gap-4">
            <UFormField label="Priority" name="priority">
              <USelect
                v-model="form.priority"
                :items="[
                  { label: '🔴 High', value: 'high' },
                  { label: '🟡 Medium', value: 'medium' },
                  { label: '🟢 Low', value: 'low' },
                ]"
                class="w-full"
              />
            </UFormField>
            <UFormField label="Due Date" name="dueDate">
              <AppDatePicker v-model="form.dueDate" placeholder="Pick due date" class="w-full" />
            </UFormField>
          </div>
          <div class="flex justify-end gap-3 pt-2 border-t border-gray-200">
            <UButton variant="ghost" color="neutral" @click="showModal = false">Cancel</UButton>
            <UButton type="submit" :loading="loading">{{ editingTask ? 'Save changes' : 'Create task' }}</UButton>
          </div>
        </UForm>
      </template>
    </UModal>
  </div>
</template>
