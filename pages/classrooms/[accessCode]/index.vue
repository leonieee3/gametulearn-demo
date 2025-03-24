<template>
  <div v-if="loading" class="p-4">
    <p>Klassenraum lädt...</p>
  </div>

  <!-- Display classroom details based on the user role -->
  <div v-else-if="classroom && isTeacher">
    <ClassroomDetailTeacher
      :classroom="classroom"
      :game="game"
      :students="students"
    />
  </div>
  <div v-else-if="classroom && !isTeacher">
    <ClassroomDetailStudent :classroom="classroom" :game="game" />
  </div>

  <div v-else class="p-4">
    <p>Klassenraum wurde nicht gefunden.</p>
  </div>
</template>

<script setup>
import ClassroomDetailTeacher from "~/components/Classroom/TeacherView/ClassroomDetailTeacher.vue";
import ClassroomDetailStudent from "~/components/Classroom/StudentView/ClassroomDetailStudent.vue";

const classroom = ref(null);
const loading = ref(true);
const isTeacher = ref(false);
const students = ref([]);
const route = useRoute();
const supabase = useSupabaseClient();
const user = useSupabaseUser();
const game = ref(null);

// load the game data based on the game id
const loadGame = async (gameId) => {
  try {
    const { data: gameData, error } = await supabase
      .from("games")
      .select("*")
      .eq("id", gameId)
      .single();

    if (error) throw error;

    game.value = gameData;
  } catch (error) {
    console.error("Error Loading game data", error);
  }
};

// Load classroom and student data based on the access code
const loadData = async () => {
  const accessCode = route.params.accessCode;

  try {
    const { data, error: accessCodeError } = await supabase
      .from("classrooms")
      .select("*")
      .eq("access_code", accessCode)
      .single();

    if (accessCodeError) throw accessCodeError;
    classroom.value = data;

    const { data: studentData, error: studentError } = await supabase.rpc(
      "get_students_for_classroom",
      { p_classroom_id: classroom.value.id },
    );

    if (studentError) throw studentError;

    students.value = studentData.map((item) => ({
      id: item.student_id,
      name: item.student_name,
    }));

    if (classroom.value.game_id) {
      loadGame(classroom.value.game_id);
    }
  } catch (error) {
    console.error("Error loading data:", error);
  } finally {
    loading.value = false;
  }
};

// Load classroom data and student list
onMounted(async () => {
  if (user.value && user.value.user_metadata.user_role === "teacher") {
    isTeacher.value = true;
  }
  loadData();
});

watchEffect(() => {
  if (!user.value) {
    return navigateTo("/");
  }
});
</script>
