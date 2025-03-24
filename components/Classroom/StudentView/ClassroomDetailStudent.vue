<template>
  <!-- Classroom details -->
  <div class="p-4">
    <div class="mb-6">
      <h1 class="text-2xl font-semibold">
        Willkommen im Klassenraum: {{ classroom.name }}
      </h1>
      <p v-if="classroom.description" class="text-gray-700 dark:text-gray-300">
        <strong>Beschreibung:</strong> {{ classroom.description }}
      </p>
    </div>

    <div class="grid grid-cols-1 gap-6 sm:grid-cols-3">
      <!-- Game details -->
      <div v-if="game" class="relative col-span-1 rounded-lg border p-4">
        <div
          class="mb-6 flex flex-col items-start sm:flex-row sm:justify-between"
        >
          <!-- Start game by klicking the link -->
          <h2 class="mb-2 text-xl font-semibold sm:mb-0">{{ game.name }}</h2>
          <NuxtLink
            :to="`/classrooms/${classroom.access_code}/playGame/${game.id}`"
            class="mt-2 inline-flex h-10 items-center justify-center rounded bg-blue-600 px-2 text-white transition hover:bg-blue-700 sm:mt-0"
          >
            <PlayIcon class="mr-2 scale-150 transform transition-transform" />
            <span>Spiel starten</span>
          </NuxtLink>
        </div>

        <!-- Game image -->
        <div class="mt-4 flex flex-col justify-between rounded-lg">
          <img
            :src="game.title_image_url"
            alt="Game Image"
            class="h-auto max-w-full rounded-lg object-contain shadow-md"
          />
        </div>
      </div>

      <!-- Insights for student -->
      <div
        v-if="student && game"
        class="col-span-1 rounded-lg border p-4 sm:col-span-2"
      >
        <StudentInsights
          :student_id="student.id"
          :student_name="student.name"
          :game_id="game.id"
          :classroom_id="classroom.id"
        />
      </div>
    </div>
  </div>
</template>

<script setup>
import PlayIcon from "vue-material-design-icons/Play.vue";
import StudentInsights from "~/components/Classroom/StudentView/StudentInsights.vue";

defineProps({
  classroom: {
    type: Object,
    required: true,
  },
  game: {
    type: Object,
    required: false,
  },
});

const user = useSupabaseUser();
const supabase = useSupabaseClient();
const student = ref(null);

// Fetch student data
async function getStudent(userId) {
  try {
    const { data, error } = await supabase.rpc("get_user", { user_id: userId });

    student.value = data[0];
    return data;
  } catch {
    console.error("Error fetching student:", error);
    student.value = null;
  }
}

onMounted(() => {
  getStudent(user.value.id);
});

watchEffect(() => {
  if (!user.value) {
    return navigateTo("/");
  }
});
</script>
