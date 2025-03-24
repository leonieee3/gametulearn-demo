<template>
  <div
    class="overflow-hidden rounded-lg bg-white p-6 shadow-md dark:bg-gray-800"
  >
    <div class="relative mb-4 h-48 w-full">
      <NuxtLink :to="'/games/' + game.id" class="mb-4 block">
        <img
          :src="game.title_image_url"
          alt="Game Image"
          class="absolute inset-0 h-full w-full rounded-md object-contain"
        />
      </NuxtLink>
    </div>
    <NuxtLink
      :to="'/games/' + game.id"
      class="mb-2 block text-2xl font-semibold text-gray-900 hover:underline dark:text-gray-100"
    >
      {{ game.name }}
    </NuxtLink>

    <p class="mb-4 line-clamp-3 text-gray-600 dark:text-gray-400">
      {{ game.description }}
    </p>

    <div class="flex items-center space-x-2">
      <NuxtRating :read-only="true" :rating-value="game.rating" />
      <p class="text-sm text-gray-500">({{ game.rating_count }} Bewertungen)</p>
    </div>
  </div>
</template>

<script setup>
defineProps({
  game: Object,
});
</script>
