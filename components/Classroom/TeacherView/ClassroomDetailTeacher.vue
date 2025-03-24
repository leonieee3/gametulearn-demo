<template>
  <div class="grid grid-cols-1 gap-6 p-6 lg:grid-cols-9">
    <div
      class="col-span-1 rounded-xl bg-white p-6 shadow-lg dark:bg-gray-800 lg:col-span-4"
    >
      <!-- Classroom details -->
      <h1 class="text-2xl font-semibold text-gray-900 dark:text-white">
        Willkommen im Klassenraum {{ classroom.name }}
      </h1>
      <p
        v-if="classroom.description"
        class="mt-2 text-gray-700 dark:text-gray-300"
      >
        <strong>Beschreibung:</strong> {{ classroom.description }}
      </p>
      <p class="mt-1 text-gray-700 dark:text-gray-300">
        <strong>Zugangscode:</strong> {{ classroom.access_code }}
      </p>
      <p class="mt-1 text-gray-700 dark:text-gray-300">
        <strong>Anzahl der Schüler:</strong> {{ classroom.student_count }}
      </p>

      <!-- Display game link if game exists -->
      <p
        v-if="game"
        class="mt-1 flex items-center text-gray-700 dark:text-gray-300"
      >
        <strong>Spiel: </strong>
        <NuxtLink
          :to="`/classrooms/${classroom.access_code}/playGame/${game.id}`"
          class="ml-1 text-blue-600 hover:underline dark:text-blue-400"
        >
          {{ game.name }}
        </NuxtLink>
      </p>

      <!-- Search bar to filter students -->
      <div class="mt-6 flex items-center">
        <SearchIcon class="mr-2 text-gray-600 dark:text-gray-300" />
        <input
          type="text"
          id="search"
          v-model="search"
          class="w-full rounded-lg border p-2 text-sm shadow-sm focus:ring-2 focus:ring-blue-500 dark:border-gray-600 dark:bg-gray-700 dark:text-white"
          placeholder="Schüler suchen..."
        />
      </div>

      <!-- Filter and sorting options -->
      <div class="mt-4 flex space-x-4">
        <div class="w-1/2">
          <label
            class="block text-sm font-medium text-gray-700 dark:text-gray-300"
            >Filtern:</label
          >
          <select
            id="filter"
            v-model="selectedFilter"
            class="mt-1 w-full rounded-lg border p-2 text-sm shadow-sm dark:border-gray-600 dark:bg-gray-700 dark:text-white"
          >
            <option value="all">Alle Schüler</option>
            <option value="active">Aktive Nutzer</option>
            <option value="inactive">Inaktive Nutzer</option>
          </select>
        </div>

        <div class="w-1/2">
          <label
            class="block text-sm font-medium text-gray-700 dark:text-gray-300"
            >Sortieren nach:</label
          >
          <select
            id="sort"
            v-model="selectedSort"
            class="mt-1 w-full rounded-lg border p-2 text-sm shadow-sm dark:border-gray-600 dark:bg-gray-700 dark:text-white"
          >
            <option value="name">Name</option>
            <option value="score">Punktzahl</option>
            <option value="level">Level</option>
            <option value="overall_playtime">Spielzeit</option>
          </select>
        </div>
      </div>

      <!-- Student list displayed as a table -->
      <div class="mt-4 overflow-x-auto">
        <table class="w-full table-auto border-collapse">
          <thead>
            <tr class="bg-gray-200 text-sm dark:bg-gray-700 dark:text-white">
              <th class="w-6 border px-2 py-1 text-left">#</th>
              <th class="border px-2 py-1 text-left">Name</th>
              <th class="border px-2 py-1 text-left">Punktzahl</th>
              <th class="border px-2 py-1 text-left">Level</th>
              <th class="border px-2 py-1 text-left">Spielzeit</th>
              <th class="border px-2 py-1"></th>
            </tr>
          </thead>
          <tbody>
            <tr
              v-for="(student, index) in sortedStudents"
              :key="student.id"
              class="cursor-pointer transition hover:bg-blue-100 dark:hover:bg-gray-600"
              :class="{
                'bg-blue-50 dark:bg-gray-500':
                  selectedStudent && selectedStudent.id === student.id,
              }"
              @click="selectStudent(student)"
            >
              <td class="border px-2 py-1">{{ index + 1 }}</td>
              <td class="border px-2 py-1">{{ student.name }}</td>
              <td class="border px-2 py-1">{{ student.score ?? "N/A" }}</td>
              <td class="border px-2 py-1">{{ student.level ?? "N/A" }}</td>
              <td class="border px-2 py-1">
                {{ student.total_playtime ?? "N/A" }}
              </td>
              <td class="border-none px-2 py-1 text-center">
                <button
                  @click="
                    deleteStudent(student.id);
                    $event.stopPropagation();
                  "
                  class="rounded-lg px-3 py-2 text-black transition hover:bg-gray-400 dark:text-gray-300"
                >
                  <DeleteIcon
                    class="scale-150 transform transition-transform"
                  />
                </button>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>

    <!-- Show detailed insights if a student is selected -->
    <div
      v-if="selectedStudent"
      class="col-span-1 rounded-xl bg-white p-6 shadow-lg dark:bg-gray-800 lg:col-span-5"
    >
      <StudentInsights
        :student_id="selectedStudent.id"
        :student_name="selectedStudent.name"
        :game_id="game.id"
        :classroom_id="classroom.id"
      />
    </div>

    <!-- Show general insights if no student is selected -->
    <div
      v-else
      class="col-span-1 rounded-xl bg-white p-6 shadow-lg dark:bg-gray-800 lg:col-span-5"
    >
      <ClassInsights
        v-if="game && studentsWithMetrics"
        :classroom_id="classroom.id"
        :game_id="game.id"
        :classroom_name="classroom.name"
      />
    </div>
  </div>
</template>

<script setup>
import ClassInsights from "@/components/Classroom/TeacherView/ClassInsights.vue";
import StudentInsights from "@/components/Classroom/StudentView/StudentInsights.vue";
import DeleteIcon from "vue-material-design-icons/Delete.vue";
import SearchIcon from "vue-material-design-icons/Magnify.vue";

const props = defineProps({
  classroom: {
    type: Object,
    required: true,
  },
  game: {
    type: Object,
    required: false,
  },
  students: {
    type: Array,
    required: true,
  },
});

const selectedFilter = ref("all");
const selectedSort = ref("name");
const search = ref("");
const selectedStudent = ref(null);
const supabase = useSupabaseClient();
const studentsWithMetrics = ref([]);

// Fetch student metrics (score, playtime, activity status) from the database
const fetchStudentMetrics = async () => {
  try {
    studentsWithMetrics.value = await Promise.all(
      props.students.map(async (student) => {
        // Fetch score for each student
        const { data: scoreData, error: scoreError } = await supabase.rpc(
          "get_score",
          {
            p_user_id: student.id,
            p_game_id: props.game.id,
            p_classroom_id: props.classroom.id,
          },
        );
        if (scoreError) {
          console.error(
            `Error fetching score for ${student.name}:`,
            scoreError,
          );
        }

        const { data: levelData, error: levelError } = await supabase.rpc(
          "get_level",
          {
            p_user_id: student.id,
            p_game_id: props.game.id,
            p_classroom_id: props.classroom.id,
          },
        );
        if (levelError) {
          console.error(
            `Error fetching level for ${student.name}:`,
            levelError,
          );
        }

        // Fetch total playtime for each student
        const { data: playtimeData, error: playtimeError } = await supabase.rpc(
          "get_total_playtime",
          {
            p_user_id: student.id,
            p_game_id: props.game.id,
            p_classroom_id: props.classroom.id,
          },
        );

        if (playtimeError) {
          console.error(
            `Error fetching total playtime for ${student.name}:`,
            playtimeError,
          );
        }

        // Fetch activity status for each student
        const { data: isActiveData, error: activityError } = await supabase.rpc(
          "is_student_active",
          {
            p_user_id: student.id,
            p_game_id: props.game.id,
            p_classroom_id: props.classroom.id,
          },
        );

        if (activityError) {
          console.error(
            `Error fetching activity status for ${student.name}:`,
            activityError,
          );
        }

        // Return combined student data with metrics
        return {
          ...student,
          score: scoreData,
          level: levelData,
          total_playtime: playtimeData,
          is_active: isActiveData,
        };
      }),
    );
  } catch (err) {
    console.error("Error fetching students:", err);
  }
};

// Delete a student from the classroom
const deleteStudent = async (studentId) => {
  try {
    const confirmDelete = confirm(
      "Möchtest du diesen Schüler wirklich aus dem Klassenraum entfernen?",
    );
    if (!confirmDelete) return;

    const { data, error } = await supabase.rpc(
      "delete_student_from_classroom",
      {
        p_student_id: studentId,
        p_classroom_id: props.classroom.id,
      },
    );

    if (error) {
      console.error("Fehler beim Löschen des Schülers:", error);
      return;
    }

    studentsWithMetrics.value = studentsWithMetrics.value.filter(
      (student) => student.id !== studentId,
    );
  } catch (error) {
    console.error("Fehler beim Löschen des Schülers:", error);
  }
};

// Computed property to sort and filter students
const sortedStudents = computed(() => {
  let filteredStudents = studentsWithMetrics.value;

  // Apply search filter
  if (search.value) {
    selectedFilter.value = "all";
    filteredStudents = filteredStudents.filter((student) =>
      student.name.toLowerCase().includes(search.value.toLowerCase()),
    );
  }

  // Apply filter based on active/inactive students
  if (selectedFilter.value === "active") {
    filteredStudents = filteredStudents.filter((student) => student.is_active);
  } else if (selectedFilter.value === "inactive") {
    filteredStudents = filteredStudents.filter((student) => !student.is_active);
  }

  // Function to convert HH:MM:SS format to total seconds
  const timeToSeconds = (time) => {
    if (!time) return 0;
    const [hours, minutes, seconds] = time.split(":").map(Number);
    return hours * 3600 + minutes * 60 + seconds;
  };

  // Apply sorting based on selected criteria
  if (selectedSort.value === "score") {
    return [...filteredStudents].sort((a, b) => b.score - a.score);
  } else if (selectedSort.value === "level") {
    return [...filteredStudents].sort((a, b) => b.level - a.level);
  } else if (selectedSort.value === "name") {
    return [...filteredStudents].sort((a, b) => a.name.localeCompare(b.name));
  } else if (selectedSort.value === "overall_playtime") {
    return [...filteredStudents].sort(
      (a, b) =>
        timeToSeconds(b.total_playtime) - timeToSeconds(a.total_playtime),
    );
  }
  return filteredStudents;
});

// Fetch student metrics whenever the students or game props change
watchEffect(() => {
  if (props.students.length > 0 && props.game) {
    fetchStudentMetrics();
  }
});

// Select or deselect a student when clicked
const selectStudent = (student) => {
  if (selectedStudent.value?.id === student.id) {
    selectedStudent.value = null;
  } else {
    selectedStudent.value = student;
  }
};
</script>
