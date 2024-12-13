<script setup lang="ts">
import {ref, onMounted, computed, watch} from "vue";
import {useRoute} from "vue-router";
import Header from "../../components/Header.vue";
import Button from "../../components/ui/Button.vue";
import {classroomService} from "../../services/classroom.service";
import {lessonService} from "../../services/lesson.service";
import {gradeService} from "../../services/grade.service";
import {userService} from "../../services/user.service";
import {gameService, type Game} from "../../services/game.service";
import BackButton from "../../components/ui/BackButton.vue";

interface ClassroomDetails {
  id: string;
  name: string;
  admins: User[];
  students: User[];
}

interface User {
  id: string;
  username: string;
  name?: string;
  surname?: string;
}

interface Lesson {
  id: string;
  name: string;
  description?: string;
  gameIds?: string[];
}

interface Grade {
  studentId: string;
  lessonId: string;
  grade: number;
  comment?: string;
}

interface AvailableStudent {
  id: string;
  username: string;
  name?: string;
  surname?: string;
}

interface Message {
  type: "success" | "error";
  text: string;
}

interface StudentDetails {
  id: string;
  username: string;
  name?: string;
  surname?: string;
  grades?: {
    lessonId: string;
    grade: number;
    comment?: string;
  }[];
  averageGrade?: number;
  completedLessons?: number;
  totalLessons?: number;
}

const route = useRoute();
const classroomId = route.params.id as string;

const classroom = ref<ClassroomDetails | null>(null);
const lessons = ref<Lesson[]>([]);
const grades = ref<Record<string, Grade[]>>({});
const loading = ref(true);
const error = ref("");

// Modal states
const showAddStudentModal = ref(false);
const showAddLessonModal = ref(false);
const showStudentDetailsModal = ref(false);

// Form states
const newLessonName = ref("");
const newLessonDescription = ref("");
const selectedGameIds = ref<string[]>([]);
const selectedStudentIds = ref<string[]>([]);
const availableStudents = ref<AvailableStudent[]>([]);
const searchQuery = ref("");
const loadingStudents = ref(false);

const message = ref<Message | null>(null);

const availableGames = ref<Game[]>([]);
const loadingGames = ref(false);

const selectedStudent = ref<StudentDetails | null>(null);

const loadClassroomData = async () => {
  try {
    loading.value = true;
    error.value = "";

    // Load classroom details
    const classroomData = await classroomService.getClassroomDetails(
      classroomId
    );
    classroom.value = classroomData;

    // Load lessons
    const lessonsData = await lessonService.getClassroomLessons(classroomId);
    lessons.value = lessonsData;

    // Load grades
    const gradesData = await gradeService.getClassroomGrades(classroomId);
    grades.value = gradesData;
  } catch (err) {
    console.error("Failed to load classroom data:", err);
    error.value =
      err instanceof Error ? err.message : "Failed to load classroom data";
  } finally {
    loading.value = false;
  }
};

const handleAddLesson = async () => {
  try {
    if (!newLessonName.value.trim()) return;

    await lessonService.createLesson({
      name: newLessonName.value.trim(),
      description: newLessonDescription.value.trim() || undefined,
      classroomId,
      gameIds:
        selectedGameIds.value.length > 0 ? selectedGameIds.value : undefined,
    });

    // Refresh lessons
    const lessonsData = await lessonService.getClassroomLessons(classroomId);
    lessons.value = lessonsData;

    // Show success message
    message.value = {type: "success", text: "Урок успешно создан"};
    setTimeout(() => {
      message.value = null;
    }, 3000);

    // Reset form and close modal
    newLessonName.value = "";
    newLessonDescription.value = "";
    selectedGameIds.value = [];
    showAddLessonModal.value = false;
  } catch (err) {
    console.error("Failed to add lesson:", err);
    error.value = err instanceof Error ? err.message : "Failed to add lesson";
    message.value = {type: "error", text: error.value};
  }
};

const handleAddStudents = async () => {
  try {
    await classroomService.addUsersToClassroom(classroomId, {
      userIds: selectedStudentIds.value,
    });

    // Refresh classroom data
    await loadClassroomData();

    // Reset form and close modal
    selectedStudentIds.value = [];
    showAddStudentModal.value = false;

    message.value = {type: "success", text: "Ученики успешно добавлены"};
    setTimeout(() => {
      message.value = null;
    }, 3000);
  } catch (err) {
    console.error("Failed to add students:", err);
    error.value = err instanceof Error ? err.message : "Failed to add students";
    message.value = {type: "error", text: error.value};
  }
};

const handleRemoveStudent = async (studentId: string) => {
  try {
    await classroomService.removeUsersFromClassroom(classroomId, {
      userIds: [studentId],
    });
    await loadClassroomData();

    message.value = {type: "success", text: "Ученик успешно удален"};
    setTimeout(() => {
      message.value = null;
    }, 3000);
  } catch (err) {
    console.error("Failed to remove student:", err);
    error.value =
      err instanceof Error ? err.message : "Failed to remove student";
    message.value = {type: "error", text: error.value};
  }
};

const loadAvailableStudents = async () => {
  try {
    loadingStudents.value = true;
    const students = await userService.getAllStudents();
    // Filter out students who are already in the classroom
    availableStudents.value = students.filter(
      (student) =>
        !classroom.value?.students.some(
          (existingStudent) => existingStudent.id === student.id
        )
    );
  } catch (err) {
    console.error("Failed to load available students:", err);
    error.value =
      err instanceof Error ? err.message : "Failed to load students";
  } finally {
    loadingStudents.value = false;
  }
};

watch(showAddStudentModal, (newValue) => {
  if (newValue) {
    loadAvailableStudents();
  } else {
    // Reset selection when modal closes
    selectedStudentIds.value = [];
  }
});

const filteredStudents = computed(() => {
  if (!searchQuery.value) return availableStudents.value;

  const query = searchQuery.value.toLowerCase();
  return availableStudents.value.filter(
    (student) =>
      student.username.toLowerCase().includes(query) ||
      student.name?.toLowerCase().includes(query) ||
      student.surname?.toLowerCase().includes(query)
  );
});

const loadAvailableGames = async () => {
  try {
    loadingGames.value = true;
    availableGames.value = await gameService.getAllGames();
  } catch (err) {
    console.error("Failed to load games:", err);
    error.value = err instanceof Error ? err.message : "Failed to load games";
  } finally {
    loadingGames.value = false;
  }
};

watch(showAddLessonModal, (newValue) => {
  if (newValue) {
    loadAvailableGames();
  } else {
    // Reset form when modal closes
    newLessonName.value = "";
    newLessonDescription.value = "";
    selectedGameIds.value = [];
  }
});

const handleStudentClick = async (student: User) => {
  try {
    // Get student's grades
    const studentGrades = grades.value[student.id] || [];

    // Calculate stats
    const avgGrade = studentGrades.length
      ? studentGrades.reduce((sum, g) => sum + g.grade, 0) /
        studentGrades.length
      : 0;

    const completedLessons = studentGrades.length;
    const totalLessons = lessons.value.length;

    selectedStudent.value = {
      ...student,
      grades: studentGrades,
      averageGrade: Number(avgGrade.toFixed(1)),
      completedLessons,
      totalLessons,
    };

    showStudentDetailsModal.value = true;
  } catch (err) {
    console.error("Failed to load student details:", err);
    error.value = "Failed to load student details";
  }
};

onMounted(() => {
  loadClassroomData();
});
</script>

<template>
  <div class="min-h-screen bg-gray-50">
    <Header />
    <BackButton />

    <!-- Message notification -->
    <div
      v-if="message"
      :class="[
        'fixed top-4 right-4 px-4 py-2 rounded-lg z-50 transition-all duration-300',
        message.type === 'success' ? 'bg-green-500' : 'bg-red-500',
        'text-white',
      ]">
      {{ message.text }}
    </div>

    <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-4 sm:py-8">
      <!-- Loading State -->
      <div v-if="loading" class="text-center py-8">
        <p>Loading...</p>
      </div>

      <!-- Error Message -->
      <div v-else-if="error" class="text-center py-8 text-red-600">
        {{ error }}
      </div>

      <!-- Content -->
      <template v-else-if="classroom">
        <div class="bg-white rounded-lg shadow p-4 sm:p-6">
          <!-- Header Section -->
          <div
            class="flex flex-col sm:flex-row sm:items-center justify-between mb-6">
            <h1
              class="text-2xl sm:text-3xl font-bold text-gray-900 mb-4 sm:mb-0">
              {{ classroom.name }}
            </h1>
            <div class="flex flex-col sm:flex-row gap-2">
              <Button @click="showAddStudentModal = true">
                Добавить ученика
              </Button>
              <Button @click="showAddLessonModal = true">
                Добавить урок
              </Button>
            </div>
          </div>

          <!-- Students Section -->
          <div class="mb-8">
            <h2 class="text-xl font-semibold mb-4">Ученики</h2>
            <div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4">
              <div
                v-for="student in classroom.students"
                :key="student.id"
                class="bg-gray-50 p-4 rounded-lg flex justify-between items-center cursor-pointer hover:bg-gray-100"
                @click="handleStudentClick(student)">
                <div>
                  <p class="font-medium">
                    {{ student.name }} {{ student.surname }}
                  </p>
                  <p class="text-sm text-gray-600">{{ student.username }}</p>
                </div>
                <button
                  @click.stop="handleRemoveStudent(student.id)"
                  class="text-red-600 hover:text-red-800">
                  Удалить
                </button>
              </div>
            </div>
          </div>

          <!-- Lessons Section -->
          <div class="mb-8">
            <h2 class="text-xl font-semibold mb-4">Уроки</h2>
            <div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4">
              <div
                v-for="lesson in lessons"
                :key="lesson.id"
                class="bg-gray-50 p-4 rounded-lg">
                <div class="flex flex-col h-full">
                  <div class="flex-1">
                    <h3 class="font-medium mb-2">{{ lesson.name }}</h3>
                    <p class="text-sm text-gray-600">
                      {{ lesson.description }}
                    </p>
                  </div>
                  <RouterLink :to="`/lessons/${lesson.id}`" class="mt-4">
                    <Button class="w-full">Подробнее</Button>
                  </RouterLink>
                </div>
              </div>
            </div>
          </div>

          <!-- Grades Section -->
          <div class="overflow-x-auto -mx-4 sm:mx-0">
            <div class="inline-block min-w-full align-middle">
              <h2 class="text-xl font-semibold mb-4 px-4 sm:px-0">Оценки</h2>
              <table class="min-w-full divide-y divide-gray-200">
                <thead class="bg-gray-50">
                  <tr>
                    <th
                      class="px-3 sm:px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
                      Ученик
                    </th>
                    <th
                      v-for="lesson in lessons"
                      :key="lesson.id"
                      class="px-3 sm:px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
                      {{ lesson.name }}
                    </th>
                  </tr>
                </thead>
                <tbody class="bg-white divide-y divide-gray-200">
                  <tr v-for="student in classroom.students" :key="student.id">
                    <td class="px-3 sm:px-6 py-4 whitespace-nowrap">
                      {{ student.name }} {{ student.surname }}
                    </td>
                    <td
                      v-for="lesson in lessons"
                      :key="lesson.id"
                      class="px-3 sm:px-6 py-4 whitespace-nowrap relative group">
                      <div class="relative">
                        <span>
                          {{
                            grades[student.id]?.find(
                              (g) => g.lessonId === lesson.id
                            )?.grade || "-"
                          }}
                        </span>
                        <!-- Tooltip for grade comment -->
                        <div
                          v-if="
                            grades[student.id]?.find(
                              (g) => g.lessonId === lesson.id
                            )?.comment
                          "
                          class="absolute left-1/2 -translate-x-1/2 bottom-full mb-2 px-3 py-2 bg-gray-900 text-white text-sm rounded-lg opacity-0 invisible group-hover:opacity-100 group-hover:visible transition-all duration-200 whitespace-normal min-w-[200px] z-10">
                          {{
                            grades[student.id]?.find(
                              (g) => g.lessonId === lesson.id
                            )?.comment
                          }}
                          <div
                            class="absolute left-1/2 -translate-x-1/2 top-full w-2 h-2 bg-gray-900 transform rotate-45"></div>
                        </div>
                      </div>
                    </td>
                  </tr>
                </tbody>
              </table>
            </div>
          </div>
        </div>
      </template>

      <!-- Add Student Modal -->
      <div
        v-if="showAddStudentModal"
        class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50">
        <div
          class="bg-white p-6 rounded-lg max-w-2xl w-full max-h-[80vh] flex flex-col">
          <h2 class="text-xl font-bold mb-4">Добавить учеников</h2>

          <!-- Search input -->
          <div class="mb-4">
            <input
              v-model="searchQuery"
              type="text"
              placeholder="Поиск по имени или username..."
              class="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-main-green focus:border-main-green" />
          </div>

          <!-- Students list -->
          <div class="flex-1 overflow-y-auto mb-4">
            <div v-if="loadingStudents" class="text-center py-4">
              Загрузка списка учеников...
            </div>

            <div
              v-else-if="filteredStudents.length === 0"
              class="text-center py-4">
              Нет доступных учеников
            </div>

            <div v-else class="space-y-2">
              <div
                v-for="student in filteredStudents"
                :key="student.id"
                class="flex items-center p-3 hover:bg-gray-50 rounded-lg">
                <input
                  type="checkbox"
                  :id="student.id"
                  :value="student.id"
                  v-model="selectedStudentIds"
                  class="h-4 w-4 text-main-green focus:ring-main-green border-gray-300 rounded" />
                <label :for="student.id" class="ml-3 flex-1 cursor-pointer">
                  <div class="font-medium">
                    {{ student.name }} {{ student.surname }}
                  </div>
                  <div class="text-sm text-gray-500">
                    {{ student.username }}
                  </div>
                </label>
              </div>
            </div>
          </div>

          <!-- Action buttons -->
          <div class="flex justify-between items-center">
            <div class="text-sm text-gray-600">
              Выбрано учеников: {{ selectedStudentIds.length }}
            </div>
            <div class="flex gap-3">
              <Button @click="showAddStudentModal = false" class="bg-gray-500">
                Отмена
              </Button>
              <Button
                @click="handleAddStudents"
                :disabled="selectedStudentIds.length === 0">
                Добавить выбранных
              </Button>
            </div>
          </div>
        </div>
      </div>

      <!-- Add Lesson Modal -->
      <div
        v-if="showAddLessonModal"
        class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50">
        <div class="bg-white p-6 rounded-lg max-w-2xl w-full">
          <h2 class="text-xl font-bold mb-4">Добавить урок</h2>

          <form @submit.prevent="handleAddLesson" class="space-y-4">
            <!-- Lesson Name -->
            <div>
              <label class="block text-sm font-medium text-gray-700">
                Название урока
              </label>
              <input
                v-model="newLessonName"
                type="text"
                required
                class="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-main-green focus:ring-main-green" />
            </div>

            <!-- Description -->
            <div>
              <label class="block text-sm font-medium text-gray-700">
                Описание
              </label>
              <textarea
                v-model="newLessonDescription"
                rows="3"
                class="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-main-green focus:ring-main-green"></textarea>
            </div>

            <!-- Games Selection -->
            <div>
              <label class="block text-sm font-medium text-gray-700 mb-2">
                Выберите игры для урока
              </label>

              <div v-if="loadingGames" class="text-center py-4">
                Загрузка списка игр...
              </div>

              <div v-else class="space-y-2 max-h-48 overflow-y-auto">
                <div
                  v-for="game in availableGames"
                  :key="game.id"
                  class="flex items-center p-2 hover:bg-gray-50 rounded">
                  <input
                    type="checkbox"
                    :id="game.id"
                    :value="game.id"
                    v-model="selectedGameIds"
                    class="h-4 w-4 text-main-green focus:ring-main-green border-gray-300 rounded" />
                  <label
                    :for="game.id"
                    class="ml-3 block text-sm text-gray-700 cursor-pointer">
                    {{ game.name }}
                  </label>
                </div>
              </div>

              <p class="mt-2 text-sm text-gray-500">
                Выбрано игр: {{ selectedGameIds.length }}
              </p>
            </div>

            <!-- Action Buttons -->
            <div class="flex justify-end space-x-4 mt-6">
              <Button
                type="button"
                @click="showAddLessonModal = false"
                class="bg-gray-500">
                Отмена
              </Button>
              <Button type="submit" :disabled="!newLessonName.trim()">
                Создать урок
              </Button>
            </div>
          </form>
        </div>
      </div>

      <!-- Student Details Modal -->
      <div
        v-if="showStudentDetailsModal && selectedStudent"
        class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50">
        <div
          class="bg-white p-6 rounded-lg max-w-2xl w-full max-h-[80vh] overflow-y-auto">
          <div class="flex justify-between items-start mb-6">
            <div>
              <h2 class="text-2xl font-bold text-gray-900">
                {{ selectedStudent.name }} {{ selectedStudent.surname }}
              </h2>
              <p class="text-gray-600">{{ selectedStudent.username }}</p>
            </div>
            <button
              @click="showStudentDetailsModal = false"
              class="text-gray-500 hover:text-gray-700">
              <svg
                class="w-6 h-6"
                fill="none"
                stroke="currentColor"
                viewBox="0 0 24 24">
                <path
                  stroke-linecap="round"
                  stroke-linejoin="round"
                  stroke-width="2"
                  d="M6 18L18 6M6 6l12 12" />
              </svg>
            </button>
          </div>

          <!-- Student Stats -->
          <div class="grid grid-cols-3 gap-4 mb-6">
            <div class="bg-gray-50 p-4 rounded-lg">
              <h3 class="text-sm font-medium text-gray-500">Средняя оценка</h3>
              <p class="text-2xl font-bold text-main-green">
                {{ selectedStudent.averageGrade }}
              </p>
            </div>
            <div class="bg-gray-50 p-4 rounded-lg">
              <h3 class="text-sm font-medium text-gray-500">
                Выполнено уро��ов
              </h3>
              <p class="text-2xl font-bold text-main-green">
                {{ selectedStudent.completedLessons }}/{{
                  selectedStudent.totalLessons
                }}
              </p>
            </div>
            <div class="bg-gray-50 p-4 rounded-lg">
              <h3 class="text-sm font-medium text-gray-500">Прогресс</h3>
              <p class="text-2xl font-bold text-main-green">
                {{
                  Math.round(
                    (selectedStudent.completedLessons ||
                      1 / (selectedStudent.totalLessons || 1)) * 100
                  )
                }}%
              </p>
            </div>
          </div>

          <!-- Grades Table -->
          <div class="overflow-x-auto">
            <table class="min-w-full divide-y divide-gray-200">
              <thead class="bg-gray-50">
                <tr>
                  <th
                    class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
                    Урок
                  </th>
                  <th
                    class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
                    Оценка
                  </th>
                  <th
                    class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">
                    Комментарий
                  </th>
                </tr>
              </thead>
              <tbody class="bg-white divide-y divide-gray-200">
                <tr v-for="lesson in lessons" :key="lesson.id">
                  <td class="px-6 py-4 whitespace-nowrap">{{ lesson.name }}</td>
                  <td class="px-6 py-4 whitespace-nowrap">
                    {{
                      selectedStudent.grades?.find(
                        (g) => g.lessonId === lesson.id
                      )?.grade || "-"
                    }}
                  </td>
                  <td class="px-6 py-4">
                    {{
                      selectedStudent.grades?.find(
                        (g) => g.lessonId === lesson.id
                      )?.comment || "-"
                    }}
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.bg-main-green {
  background-color: #4caf50;
}
</style>
