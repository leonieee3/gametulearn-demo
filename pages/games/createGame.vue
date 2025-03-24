<template>
  <div
    class="flex min-h-screen items-center justify-center bg-gray-100 px-4 py-10 dark:bg-gray-900"
  >
    <div
      class="w-full max-w-md rounded-lg bg-white p-6 shadow-md dark:bg-gray-800"
    >
      <h2
        class="mb-6 text-center text-2xl font-bold text-gray-900 dark:text-gray-100"
      >
        Neues Spiel erstellen
      </h2>
      <form @submit.prevent="createGame" class="space-y-4">
        <!-- Game name input-->
        <div>
          <label
            for="name"
            class="mb-1 block text-sm font-medium text-gray-700 dark:text-gray-300"
          >
            Spielname
          </label>
          <input
            id="name"
            v-model="newGame.name"
            type="text"
            required
            class="w-full rounded-lg border bg-gray-50 p-2 text-gray-900 focus:outline-none focus:ring-2 focus:ring-blue-500 dark:bg-gray-700 dark:text-gray-100"
          />
        </div>

        <!-- Description input -->
        <div>
          <label
            for="description"
            class="mb-1 block text-sm font-medium text-gray-700 dark:text-gray-300"
          >
            Beschreibung
          </label>
          <textarea
            id="description"
            v-model="newGame.description"
            required
            class="w-full rounded-lg border bg-gray-50 p-2 text-gray-900 focus:outline-none focus:ring-2 focus:ring-blue-500 dark:bg-gray-700 dark:text-gray-100"
          ></textarea>
        </div>

        <!-- Game category selection -->
        <div>
          <label
            for="category"
            class="mb-1 block text-sm font-medium text-gray-700 dark:text-gray-300"
          >
            Kategorie
          </label>
          <select
            id="category"
            v-model="newGame.category"
            class="w-full rounded-lg border bg-gray-50 p-2 text-gray-900 focus:outline-none focus:ring-2 focus:ring-blue-500 dark:bg-gray-700 dark:text-gray-100"
          >
            <option
              v-for="category in categories"
              :key="category"
              :value="category"
            >
              {{ category }}
            </option>
          </select>
        </div>

        <!-- Image upload -->
        <div>
          <label
            for="image"
            class="mb-1 block text-sm font-medium text-gray-700 dark:text-gray-300"
          >
            Laden Sie ein Foto hoch
          </label>
          <input
            id="image"
            type="file"
            @change="handleImageUpload"
            class="w-full cursor-pointer rounded-lg border border-gray-300 bg-gray-50 text-sm text-gray-900 focus:outline-none dark:bg-gray-700 dark:text-gray-100"
          />
        </div>

        <!-- Submit button -->
        <div class="pt-4">
          <button
            type="submit"
            class="w-full rounded-lg bg-blue-500 px-4 py-2 text-white shadow-md transition duration-300 hover:bg-blue-600 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-offset-2"
          >
            Spiel erstellen
          </button>
        </div>
      </form>
    </div>
  </div>
</template>

<script setup>
const supabase = useSupabaseClient();
const user = useSupabaseUser();
const router = useRouter();
const imageUrl = ref("");

const categories = ref([]);
const newGame = ref({
  name: "",
  description: "",
  category: "",
});

// Fetch categories from the database
const fetchCategories = async () => {
  try {
    const { data, error } = await supabase.rpc("get_game_genres");
    if (error) throw error;
    categories.value = data;
  } catch (error) {
    console.error("Error loading categories:", error);
  }
};

onMounted(fetchCategories);

// Handle image upload to Supabase Storage
const handleImageUpload = async (event) => {
  const file = event.target.files[0];
  if (!file) return;

  try {
    const { data, error } = await supabase.storage
      .from("game_images")
      .upload(file.name, file);

    if (error) throw error;

    const { data: urlData, error: urlError } = supabase.storage
      .from("game_images")
      .getPublicUrl(file.name);

    if (urlError) throw urlError;

    imageUrl.value = urlData.publicUrl;
    console.log("Image uploaded successfully! ", imageUrl.value);
  } catch (error) {
    console.error("Error uploading image:", error);
  }
};

// Create a new game in the database
const createGame = async () => {
  try {
    const { data, error } = await supabase.from("games").insert([
      {
        ...newGame.value,
        creator_id: user.value.id,
        created_at: new Date(),
        title_image_url: imageUrl.value,
      },
    ]);

    if (error) throw error;

    alert("Game created successfully!");
    newGame.value = {
      name: "",
      description: "",
      category: "",
      rating: 0,
      title_image_url: "",
    };

    router.push("/games");
  } catch (error) {
    console.error("Error creating game:", error);
  }
};

// Redirect to login page if user is not logged in
watchEffect(() => {
  if (!user.value) {
    navigateTo("/login");
  }
});
</script>
