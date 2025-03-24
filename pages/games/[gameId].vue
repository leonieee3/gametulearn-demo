<template>
  <!-- Game details section -->
  <div
    v-if="game"
    class="game-details min-h-screen bg-gray-100 p-8 text-gray-900 dark:bg-gray-900 dark:text-gray-100"
  >
    <h1 class="mb-6 text-center text-4xl font-bold">{{ game.name }}</h1>
    <div
      class="mx-auto max-w-4xl rounded-lg bg-white p-6 shadow-lg dark:bg-gray-800"
    >
      <div class="relative mb-4 h-64 w-full">
        <img
          :src="game.title_image_url"
          alt="Game Title Image"
          class="absolute inset-0 h-full w-full rounded-md object-contain"
        />
      </div>
      <p class="mb-4 text-gray-700 dark:text-gray-300">
        {{ game.description }}
      </p>
      <p class="text-sm text-gray-600 dark:text-gray-400">
        <strong>Kategorie:</strong> {{ game.category }}
      </p>

      <div
        class="flex items-center space-x-2 text-sm text-gray-600 dark:text-gray-400"
      >
        <p><strong>Bewertung:</strong> {{ game.rating }}</p>
        <NuxtRating :read-only="true" :rating-value="game.rating" />
        <p>({{ game.rating_count }} Bewertungen)</p>
      </div>

      <!-- Rating section only visible for users who are not guests -->
      <div v-if="!isGuest">
        <NuxtRating
          class="mt-2"
          v-model="userRating"
          :read-only="false"
          :rating-value="userRating"
          :rating-step="1"
          @rating-selected="setRating"
        />

        <!-- Button to confirm the rating -->
        <button
          :disabled="!userRating"
          @click="onRate"
          class="mt-2 rounded bg-blue-500 px-4 py-2 text-white hover:bg-blue-600"
        >
          {{ buttonText }}
        </button>
      </div>
      <!-- Message for guest users -->
      <div v-else class="text-grey-500 mt-4">
        (Du musst dich anmelden, um eine Bewertung abzugeben.)
      </div>
    </div>
  </div>
  <div v-else class="mt-10 text-center">
    <p class="text-gray-500 dark:text-gray-400">Lade die Game Details...</p>
  </div>
</template>

<script setup>
const supabase = useSupabaseClient();
const user = useSupabaseUser();
const route = useRoute();
const game = ref(null);

const loggedIn = ref(false);
const userRating = ref(0);
const previousRating = ref(null);
const buttonText = computed(() =>
  previousRating.value ? "Bewertung ändern" : "Bewertung abgeben",
);
const isGuest = computed(() => user.value?.user_metadata.user_role === "guest");

// Fetch the game data and user's previous rating on component mount
onMounted(async () => {
  const gameId = route.params.gameId;

  try {
    // Fetch the game data
    const { data, error: gameError } = await supabase
      .from("games")
      .select("*")
      .eq("id", gameId)
      .single();

    if (gameError) throw gameError;

    game.value = data;

    // Fetch the user's previous rating if logged in
    if (loggedIn.value) {
      const { data: ratingData, error: ratingError } = await supabase
        .from("ratings")
        .select("rating")
        .eq("user_id", user.value.id)
        .eq("game_id", game.value.id)
        .single();

      if (ratingError) throw ratingError;
      else if (ratingData) {
        previousRating.value = ratingData.rating;
        userRating.value = ratingData.rating; // Set previous rating to userRating
      }
    }
  } catch (error) {
    console.error("Error loading game details:", error);
  }
});

// Set the rating value when the user selects a star
const setRating = (rating) => {
  userRating.value = rating;
};

// Event handler for the rating button click
const onRate = async () => {
  if (!loggedIn.value) {
    alert("Please log in to submit a rating.");
    return;
  }
  await addRating(userRating.value);
  await fetchUpdatedRating();
};

// Add or update the rating in Supabase
const addRating = async (newRating) => {
  try {
    const { data, error } = await supabase.from("ratings").upsert(
      [
        {
          user_id: user.value.id,
          game_id: game.value.id,
          rating: newRating,
        },
      ],
      { onConflict: ["user_id", "game_id"] },
    );

    if (error) throw error;
  } catch (error) {
    console.error("Error adding/updating rating:", error);
  }
};

// Fetch the updated game rating and rating count from Supabase
const fetchUpdatedRating = async () => {
  try {
    const { data, error } = await supabase
      .from("games")
      .select("rating, rating_count")
      .eq("id", game.value.id)
      .single();

    if (error) throw error;

    if (data) {
      game.value.rating = data.rating;
      game.value.rating_count = data.rating_count;
    }
  } catch (error) {
    console.error("Error fetching updated rating:", error);
  }
};

watchEffect(() => {
  loggedIn.value = user.value !== null;
});
</script>
