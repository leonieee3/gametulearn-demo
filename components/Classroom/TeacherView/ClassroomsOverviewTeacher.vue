<template>
  <div class="mx-auto max-w-4xl px-4 py-8">
    <!-- Header with button -->
    <div class="mb-6 flex items-center justify-between">
      <h1 class="text-2xl font-bold text-gray-900 dark:text-gray-100">
        Deine Klassenräume
      </h1>
      <button
        @click="openCreateClassroomModal"
        class="flex items-center rounded-lg bg-blue-500 px-4 py-2 font-medium text-white shadow-md transition duration-300 hover:bg-blue-600"
      >
        <plus-icon class="mr-2 transform transition-transform" />
        Klassenraum erstellen
      </button>
    </div>

    <!-- Classroom list -->
    <div v-if="classrooms.length === 0" class="py-8 text-center text-gray-600">
      <p>Du hast noch keine Klassenräume erstellt.</p>
    </div>
    <div v-else>
      <ul class="space-y-4">
        <li
          v-for="classroom in classrooms"
          :key="classroom.id"
          class="flex items-center justify-between rounded-lg bg-white p-4 shadow transition-shadow hover:shadow-lg dark:bg-gray-800"
        >
          <div>
            <!-- Classroom name as link o classroom -->
            <NuxtLink
              :to="`/classrooms/${classroom.access_code}`"
              class="text-lg font-semibold text-gray-800 hover:underline dark:text-gray-100"
            >
              {{ classroom.name }}
            </NuxtLink>
            <p class="text-sm text-gray-600 dark:text-gray-400">
              {{ classroom.description }}
            </p>

            <NuxtLink
              v-if="games.find((g) => g.id === classroom.game_id)"
              :to="`/games/${classroom.game_id}`"
              class="text-sm italic text-gray-600 hover:underline dark:text-gray-400"
            >
              {{ games.find((g) => g.id === classroom.game_id)?.name }}
            </NuxtLink>
            <span v-else class="text-sm text-gray-600 dark:text-gray-400">
              Unbekanntes Spiel
            </span>
          </div>
          <div class="flex items-center space-x-2">
            <!-- Delete button with icon -->
            <button
              @click="deleteClassroom(classroom.id)"
              class="rounded-lg px-3 py-2 text-black transition hover:bg-gray-400 dark:text-gray-300"
            >
              <delete-icon class="scale-150 transform transition-transform" />
            </button>
          </div>
        </li>
      </ul>
    </div>

    <!-- Create classroom modal -->
    <div
      v-if="classroomModalOpen"
      class="fixed inset-0 z-50 flex items-center justify-center bg-black bg-opacity-50"
    >
      <div
        class="w-full max-w-md rounded-lg bg-white p-6 shadow-lg dark:bg-gray-800"
      >
        <h2 class="mb-4 text-xl font-bold text-gray-900 dark:text-gray-100">
          Neuen Klassenraum erstellen
        </h2>
        <form @submit.prevent="createClassroom" class="space-y-4">
          <div>
            <label
              for="name"
              class="block text-sm font-medium text-gray-700 dark:text-gray-300"
            >
              Name des Klassenraums
            </label>
            <input
              id="name"
              v-model="newClassroom.name"
              type="text"
              required
              class="mt-1 block w-full rounded-md border bg-gray-50 p-2 text-gray-900 focus:border-blue-500 focus:ring-blue-500 dark:bg-gray-700 dark:text-gray-100"
            />
          </div>
          <div>
            <label
              for="description"
              class="block text-sm font-medium text-gray-700 dark:text-gray-300"
            >
              Beschreibung
            </label>
            <textarea
              id="description"
              v-model="newClassroom.description"
              class="mt-1 block w-full rounded-md border bg-gray-50 p-2 text-gray-900 focus:border-blue-500 focus:ring-blue-500 dark:bg-gray-700 dark:text-gray-100"
            ></textarea>
          </div>

          <div>
            <label
              for="game"
              class="block text-sm font-medium text-gray-700 dark:text-gray-300"
            >
              Spiel auswählen
            </label>
            <select
              id="game"
              v-model="newClassroom.selectedGame"
              required
              class="mt-1 block w-full rounded-md border bg-gray-50 p-2 text-gray-900 focus:border-blue-500 focus:ring-blue-500 dark:bg-gray-700 dark:text-gray-100"
            >
              <option value="" disabled>Bitte ein Spiel auswählen</option>
              <option v-for="game in games" :key="game.id" :value="game.id">
                {{ game.name }}
              </option>
            </select>
          </div>

          <div class="flex justify-end space-x-2">
            <button
              type="button"
              @click="closeClassroomModal"
              class="rounded-lg bg-gray-300 px-4 py-2 text-gray-800 transition hover:bg-gray-400 dark:bg-gray-600 dark:text-gray-100 dark:hover:bg-gray-700"
            >
              Abbrechen
            </button>
            <button
              type="submit"
              class="rounded-lg bg-blue-500 px-4 py-2 text-white shadow-md transition hover:bg-blue-600"
            >
              Erstellen
            </button>
          </div>
        </form>
      </div>
    </div>
  </div>
</template>

<script setup>
import DeleteIcon from "vue-material-design-icons/Delete.vue";
import PlusIcon from "vue-material-design-icons/Plus.vue";

const accessCode = ref(null);
const classrooms = ref([]);
const classroomModalOpen = ref(false);
const newClassroom = ref({
  name: "",
  description: "",
  selectedGame: "",
});
const games = ref([]);
const supabase = useSupabaseClient();
const user = useSupabaseUser();

watch(classrooms, loadGames, { immediate: true });

// open the modal to create a new classroom
function openCreateClassroomModal() {
  classroomModalOpen.value = true;
  loadGames();
}

// close the modal to create a new classroom
function closeClassroomModal() {
  classroomModalOpen.value = false;
  newClassroom.value.name = "";
  newClassroom.value.description = "";
  accessCode.value = null;
}

// load all games from the database to select a game for the classroom
async function loadGames() {
  try {
    const { data, error } = await supabase.from("games").select("*");

    if (error) throw error;

    games.value = data;
  } catch (error) {
    console.error("Error loading games:", error);
  }
}

// create a new classroom
async function createClassroom() {
  try {

    const { data, error } = await supabase.from("classrooms").insert([
      {
        created_by: user.value.id,
        name: newClassroom.value.name,
        description: newClassroom.value.description,
        game_id: newClassroom.value.selectedGame,
      },
    ]);

    if (error) throw error;

    await fetchClassrooms();
    closeClassroomModal();
  } catch (error) {
    console.error("Error creating classroom:", error);
  }
}

// fetch all classrooms created by the user
async function fetchClassrooms() {
  try {
    const { data, error } = await supabase
      .from("classrooms")
      .select("*")
      .eq("created_by", user.value.id);

    if (error) throw error;

    classrooms.value = data;
  } catch (error) {
    console.error("Error fetching classrooms:", error);
  }
}

// delete a classroom
async function deleteClassroom(classroomId) {
  try {
    const confirmDelete = confirm(
      "Möchtest du diesen Klassenraum wirklich löschen?",
    );
    if (!confirmDelete) return;

    const { data, error } = await supabase
      .from("classrooms")
      .delete()
      .eq("id", classroomId);

    if (error) throw error;

    classrooms.value = classrooms.value.filter(
      (classroom) => classroom.id !== classroomId,
    );
  } catch (error) {
    console.error("Error deleting classroom:", error);
  }
}

onMounted(fetchClassrooms);

watchEffect(() => {
  if (!user.value) {
    navigateTo("/");
  }
});
</script>
