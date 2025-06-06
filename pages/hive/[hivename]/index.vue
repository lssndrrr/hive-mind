<template>
    <div
        :class="[
            'grid gap-6 p-4 md:p-6 h-screen w-screen overflow-hidden',
            isTaskDetailsOpen || isMemberDetailsOpen
                ? 'grid-cols-[3fr_1fr]'
                : 'grid-cols-1', // Dynamic columns
        ]"
    >
        <div
            :class="[
                'flex flex-col space-y-8 overflow-y-auto main-content-scroll',
                !isTaskDetailsOpen && !isMemberDetailsOpen
                    ? 'col-span-full'
                    : '', // Takes full width if details are closed
            ]"
        >
            <div>
                <div class="flex items-center space-x-2 pt-8">
                    <Icon
                        name="material-symbols:hive-rounded"
                        class="h-6 w-6 bg-[#A86523]"
                    />
                    <h1 class="text-xl font-bold text-[#A86523]">
                        {{ currentHive?.name || 'Hive Dashboard' }}
                    </h1>
                </div>
            </div>

            <UTabs :items="items" class="w-[90%]" variant="link">
                <template #content="{ item }">
                    <div v-if="item.label === 'Dashboard'" class="space-y-8">
                        <div>
                            <div
                                class="flex justify-between items-center space-x-2 my-4"
                            >
                                <div class="flex items-center space-x-2">
                                    <Icon
                                        name="heroicons:check-circle-20-solid"
                                        class="h-6 w-6 text-[#A86523]"
                                    />
                                    <h2
                                        class="text-lg font-semibold text-[#A86523]"
                                    >
                                        Hive Tasks
                                    </h2>
                                </div>
                                <div>
                                    <UButton
                                        color="neutral"
                                        variant="solid"
                                        icon="heroicons:plus-20-solid"
                                        size="md"
                                        class="text-[#A86523] hover:bg-[#E9A319] hover:text-white cursor:pointer"
                                        @click="handleAddTask()"
                                    >
                                        Add Task
                                    </UButton>
                                </div>
                            </div>

                            <div class="flex space-between space-x-4 pb-2">
                                <KanbanColumn
                                    title="To Do"
                                    :tasks="todoTasks"
                                    @view-task-details="showTaskDetails"
                                />
                                <KanbanColumn
                                    title="In Progress"
                                    :tasks="inProgressTasks"
                                    @view-task-details="showTaskDetails"
                                />
                                <KanbanColumn
                                    title="Done"
                                    :tasks="doneTasks"
                                    @view-task-details="showTaskDetails"
                                />
                                <KanbanColumn
                                    title="Overdue"
                                    :tasks="overdueTasks"
                                    @view-task-details="showTaskDetails"
                                />
                            </div>
                        </div>

                        <div>
                            <div class="flex items-center space-x-2 mb-4">
                                <Icon
                                    name="mdi:calendar"
                                    class="h-6 w-6 text-[#A86523]"
                                />
                                <h2
                                    class="text-lg font-semibold text-[#A86523]"
                                >
                                    Hive Calendar
                                </h2>
                            </div>
                            <div class="w-full">
                                <Calendar
                                    :tasks="allHiveTasks"
                                    @view-task-details="showTaskDetails"
                                />
                            </div>
                        </div>
                    </div>

                    <div v-else-if="item.label === 'Members'">
                        <div class="flex items-center space-x-2 my-4">
                            <Icon
                                name="icon-park-solid:bee"
                                class="h-6 w-6 text-[#A86523]"
                            />
                            <h2 class="text-lg font-semibold text-[#A86523]">
                                Hive Members
                            </h2>
                        </div>
                        <div class="p-8">
                            <MembersTab
                                :members="hiveMembers"
                                @view-member-details="showMemberDetails"
                            />
                        </div>
                    </div>

                    <div v-else-if="item.label === 'Tasks'">
                        <div class="flex items-center space-x-2 my-4">
                            <Icon
                                name="solar:checklist-minimalistic-bold"
                                class="h-6 w-6 text-[#A86523]"
                            />
                            <h2 class="text-lg font-semibold text-[#A86523]">
                                Hive Task Overview
                            </h2>
                        </div>
                        <div class="p-8">
                            <TasksTab
                                :todo-tasks="todoTasks"
                                :in-progress-tasks="inProgressTasks"
                                :done-tasks="doneTasks"
                                :overdue-tasks="overdueTasks"
                                @view-task-details="showTaskDetails"
                                @add-task="handleAddTask"
                            />
                        </div>
                    </div>
                </template>
            </UTabs>
        </div>

        <div class="h-full overflow-hidden">
            <MemberDetails
                v-model="isMemberDetailsOpen"
                v-if="isMemberDetailsOpen && selectedMember && currentHiveId"
                :member="selectedMember"
                :hive-id="currentHiveId"
                @close="isMemberDetailsOpen = false"
            ></MemberDetails>
        </div>

        <div class="h-full overflow-hidden">
            <TaskDetails
                v-if="isTaskDetailsOpen && selectedTask"
                :task="selectedTask"
                @close="isTaskDetailsOpen = false"
                @update-status="handleUpdateTaskStatus"
                @update-date="handleUpdateTaskDate"
                class="details-panel"
            ></TaskDetails>
        </div>
    </div>

    <AddTaskModal
        :is-open="isAddTaskModalOpen"
        :task="defaultTask"
        :created-by="userStore.user?.id"
        @close="isAddTaskModalOpen = false"
        @submit="handleSubmit"
    />
</template>

<script setup lang="ts">
import { ref, computed, onMounted, watch } from 'vue'
import type { Hive, HiveMember } from '~/interfaces/hive'
import type { Task, AddTaskPayload } from '~/interfaces/task'
import type { TabsItem } from '@nuxt/ui'
import { useHiveTaskStore } from '~/stores/task'
import { useHiveStore } from '~/stores/hive'
import { useRoute } from 'vue-router'

definePageMeta({
    layout: 'hive',
    middleware: 'auth',
})

const userStore = useUserStore()
const route = useRoute()
const taskStore = useHiveTaskStore()
const hiveStore = useHiveStore()
const hiveId = hiveStore.currentHiveId

const isTaskDetailsOpen = ref(false)
const isMemberDetailsOpen = ref(false)
const isAddTaskModalOpen = ref(false)
const selectedTask = ref<Task | null>(null)
const selectedMember = ref<HiveMember | null>(null)

const defaultTask = ref<Task>({
    id: 0,
    name: '',
    description: '',
    assignee: null,
    due_date: '',
    status: 'TD',
    priority: 'M',
    hive: 1,
    created_by: 0,
    created_on: '',
})

const tasksForHive = computed(() => {
    return hiveStore.getTasksForHive(Number(route.params.hivename))
})

interface MemberOption {
    id: number | string
    name: string
}
const defaultNewTaskState = () => ({
    title: '',
    description: '',
    assignee: undefined as MemberOption | undefined, // Store assignee ID
    date: undefined as Date | undefined,
})
const newTask = ref(defaultNewTaskState())

const items = ref<TabsItem[]>([
    { label: 'Dashboard', icon: 'mdi:beehive-outline' },
    { label: 'Members', icon: 'icon-park-solid:bee' },
    { label: 'Tasks', icon: 'i-lucide-check-square' },
])

const allTasks = computed(() => {
    return currentHiveId.value
        ? hiveStore.getTasksForHive(currentHiveId.value)
        : []
})

const todoTasks = computed(() =>
    allTasks.value.filter((t) => t.status === 'TD')
)
const inProgressTasks = computed(() =>
    allTasks.value.filter((t) => t.status === 'IP')
)
const doneTasks = computed(() => allTasks.value.filter((t) => t.status === 'D'))

const overdueTasks = computed(() =>
    allTasks.value.filter((t) => t.status === 'OD')
)

const allHiveTasks = computed(() => allTasks.value) // For calendar

const hiveMembers = computed(() => {
    return currentHiveId.value ? hiveStore.getMembers(currentHiveId.value) : []
})

const currentHive = computed(() => {
    return currentHiveId.value
        ? hiveStore.hives.find((h) => h.id === currentHiveId.value)
        : null
})

const currentHiveId = computed(() => {
    const idParam = route.params.hivename || route.params.hiveId
    return idParam ? Number(idParam) : null
})

watch(
    currentHiveId,
    (newId, oldId) => {
        if (newId && newId !== oldId) {
            hiveStore.setCurrentHiveId(newId)
        }
    },
    { immediate: true }
)

onMounted(async () => {
    const hiveId = currentHiveId.value
    if (hiveId) {
        hiveStore.setCurrentHiveId(hiveId)

        // Fetch hives if not already loaded
        if (hiveStore.hives.length === 0) {
            await taskStore.fetchHiveTasks(hiveId)
        }

        // Always fetch tasks for this hive when the page is loaded
        await taskStore.fetchHiveTasks(hiveId)
        console.log('Fetched tasks for hive:', hiveStore.hiveTasks[hiveId])
        console.log('Loaded Hive Members:', hiveMembers.value)
    }
})

const memberOptions = computed(() => {
    return hiveMembers.value.map((member) => ({
        id: member.user.id,
        name: member.user.username,
    }))
})

const userTaskStore = useUserTaskStore()

const memberNameById = (id: string | number | undefined) => {
    if (!id) return ''
    return hiveMembers.value.find((m) => m.user.id === id)?.user.username || ''
}
async function handleSubmit(taskPayload: AddTaskPayload) {
    if (!hiveId) {
        console.error('Hive ID is not available')
        return // Early exit if hiveId is not set
    }

    console.log('Submitting new task:', taskPayload)

    // Ensure hiveId is part of the task payload
    const updatedTaskPayload = {
        ...taskPayload,
        hive: hiveId, // Use the current hive ID from the store
    }

    // Now call your store to create the task with the updated payload
    await hiveStore.addTask(updatedTaskPayload)

    // Optionally, fetch user tasks if needed
    // await userTaskStore.fetchUserTasks(userStore.user?.id || 10);

    console.log('User Tasks after submit:', userTaskStore.tasks)

    // Close the modal after submitting
    isAddTaskModalOpen.value = false
}

function showTaskDetails(task: Task) {
    selectedTask.value = task
    isTaskDetailsOpen.value = true
}

function showMemberDetails(member: any) {
    selectedMember.value = member
    isMemberDetailsOpen.value = true
}


function handleAddTask() {
    console.log('Handling add task...')
    newTask.value = defaultNewTaskState() // Reset form
    selectedTask.value = null
    isAddTaskModalOpen.value = true
}


function handleUpdateTaskStatus({
    taskId,
    status,
}: {
    taskId: string | number
    status: string
}) {
    console.log(
        `TODO: Call store action to update task ${taskId} status to ${status}`
    )
    // await hiveStore.updateTaskStatus(taskId, status); // Example store action call
    // Note: The computed properties will update automatically when store state changes
}

function handleUpdateTaskDate({
    taskId,
    date,
}: {
    taskId: string | number
    date: string | null
}) {
    console.log(
        `TODO: Call store action to update task ${taskId} date to: ${date}`
    )
    // await hiveStore.updateTaskDate(taskId, date); // Example store action call
}
</script>
