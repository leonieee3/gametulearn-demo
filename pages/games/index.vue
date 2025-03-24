<template>
  <div
    class="min-h-screen bg-gray-100 p-4 text-gray-900 dark:bg-gray-900 dark:text-gray-100"
  >
    <h1 class="mb-6 text-center text-4xl font-bold">Spiele</h1>

    <!-- Add New Game Button -->
    <div
      v-if="
        user?.user_metadata?.user_role === 'teacher' ||
        user?.user_metadata?.user_role === 'admin'
      "
      class="mb-4 flex justify-center"
    >
      <NuxtLink
        to="/games/createGame"
        class="flex items-center rounded-lg bg-blue-500 px-5 py-2 font-medium text-white shadow-md transition duration-300 hover:bg-blue-600"
      >
        <plus-icon class="mr-2 transform transition-transform" />
        <span class="mr-2">Spiel hinzufügen</span>
      </NuxtLink>
    </div>

    <div class="mb-6 flex items-center justify-between">
      <!-- Filter by Category -->
      <div>
        <label
          for="category"
          class="mb-2 block text-gray-700 dark:text-gray-300"
          >Filtern nach Kategorie:</label
        >
        <select
          id="category"
          v-model="filters.category"
          class="rounded bg-gray-200 p-2 transition duration-300 focus:outline-none focus:ring-2 focus:ring-blue-500 dark:bg-gray-700"
        >
          <option value="">Alle</option>
          <option
            v-for="category in categories"
            :key="category"
            :value="category"
          >
            {{ category }}
          </option>
        </select>
      </div>
      <!-- Order by name, rating or created_at-->
      <div>
        <label for="order" class="mb-2 block text-gray-700 dark:text-gray-300"
          >Sortieren nach:</label
        >
        <select
          id="order"
          v-model="order"
          class="rounded bg-gray-200 p-2 transition duration-300 focus:outline-none focus:ring-2 focus:ring-blue-500 dark:bg-gray-700"
        >
          <option value="name">Name</option>
          <option value="rating">Bewertung</option>
          <option value="created_at">Erstellt am</option>
        </select>
      </div>
    </div>
    <div class="grid grid-cols-1 gap-4 md:grid-cols-3">
      <GameCard
        v-for="game in filteredAndOrderedGames"
        :key="game.id"
        :game="game"
      />
    </div>
  </div>
</template>

<script setup>
import PlusIcon from "vue-material-design-icons/Plus.vue";

const supabase = useSupabaseClient();
const user = useSupabaseUser();
const games = ref([]); // Array to store games loaded from the database
const filters = ref({
  category: "",
});
const order = ref("name");

// Generate category options dynamically based on loaded games
const categories = computed(() => {
  return [...new Set(games.value.map((game) => game.category))];
});

// Combine filtering and sorting of games
const filteredAndOrderedGames = computed(() => {
  let filteredGames = games.value;

  if (filters.value.category) {
    filteredGames = filteredGames.filter(
      (game) => game.category === filters.value.category,
    );
  }

  return filteredGames.sort((a, b) => {
    if (order.value === "rating") {
      return b.rating - a.rating;
    }
    if (order.value === "created_at" || order.value === "updated_at") {
      return new Date(b[order.value]) - new Date(a[order.value]);
    }
    if (a[order.value] < b[order.value]) return -1;
    if (a[order.value] > b[order.value]) return 1;
    return 0;
  });
});

// Load games from the Supabase database
async function loadGames() {
  try {
    const { data, error } = await supabase.from("games").select("*");
    if (error) throw error;
    games.value = data; // Assign loaded games to the games array
  } catch (error) {
    console.error("Error loading games:", error.message);
  }
}

// Load games when the page is accessed
onMounted(() => {
  loadGames();
});
</script>
