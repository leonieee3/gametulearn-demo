<template>
  <div
    class="flex min-h-screen flex-col bg-gray-100 text-gray-900 dark:bg-gray-900 dark:text-gray-100"
  >
    <!-- Header -->
    <header
      class="flex items-center justify-between bg-white px-8 py-4 shadow-md dark:bg-gray-800"
    >
      <div class="flex items-center space-x-10">
        <NuxtLink to="/" class="h-12">
          <img src="/logo.png" alt="Logo" class="h-12 cursor-pointer" />
        </NuxtLink>
        <nav class="hidden space-x-10 md:flex">
          <NuxtLink
            to="/games"
            class="transition duration-300 hover:text-blue-600 dark:hover:text-blue-400"
            :class="{
              'font-bold text-blue-600 dark:text-blue-400':
                $route.path === '/games',
            }"
          >
            Spiele
          </NuxtLink>
          <NuxtLink
            to="/about"
            class="transition duration-300 hover:text-blue-600 dark:hover:text-blue-400"
            :class="{
              'font-bold text-blue-600 dark:text-blue-400':
                $route.path === '/about',
            }"
          >
            Über uns
          </NuxtLink>
          <NuxtLink
            to="/classrooms"
            v-if="loggedIn"
            class="transition duration-300 hover:text-blue-600 dark:hover:text-blue-400"
            :class="{
              'font-bold text-blue-600 dark:text-blue-400':
                $route.path === '/classrooms',
            }"
          >
            Klassenräume
          </NuxtLink>
        </nav>
      </div>

      <!-- Menu for Small Screens -->
      <div class="flex items-center space-x-4 md:hidden">
        <button @click="toggleMenu" class="text-gray-600 dark:text-gray-400">
          <svg
            xmlns="http://www.w3.org/2000/svg"
            fill="none"
            viewBox="0 0 24 24"
            stroke="currentColor"
            class="h-6 w-6"
          >
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              stroke-width="2"
              d="M4 6h16M4 12h16M4 18h16"
            />
          </svg>
        </button>
      </div>

      <!-- User Action Links (Login, Account, etc.) -->
      <div class="flex items-center space-x-4">
        <NuxtLink
          to="/login"
          v-if="!loggedIn"
          class="flex items-center space-x-2 rounded bg-blue-500 p-2 text-white transition duration-300 hover:bg-blue-600"
        >
          <LoginIcon class="h-5 w-5" />
          <span>Login</span>
        </NuxtLink>
        <NuxtLink
          to="/account"
          v-if="loggedIn"
          class="flex items-center space-x-2 rounded bg-blue-500 p-2 text-white transition duration-300 hover:bg-blue-600"
        >
          <AccountIcon class="h-5 w-5" />
          <span>Konto</span>
        </NuxtLink>
        <button
          class="flex items-center space-x-2 rounded bg-gray-200 p-3 transition duration-300 hover:bg-gray-300 dark:bg-gray-700 dark:hover:bg-gray-600"
          @click="toggleDarkMode"
        >
          <ThemeLightDark class="h-5 w-5 scale-150" />
        </button>
      </div>
    </header>

    <!-- Mobile Menu -->
    <div
      v-if="menuOpen"
      class="space-y-4 bg-gray-200 px-6 py-4 dark:bg-gray-800 md:hidden"
    >
      <NuxtLink
        to="/games"
        class="block transition duration-300 hover:text-blue-600 dark:hover:text-blue-400"
        :class="{
          'font-bold text-blue-600 dark:text-blue-400':
            $route.path === '/games',
        }"
      >
        Spiele
      </NuxtLink>
      <NuxtLink
        to="/about"
        class="block transition duration-300 hover:text-blue-600 dark:hover:text-blue-400"
        :class="{
          'font-bold text-blue-600 dark:text-blue-400':
            $route.path === '/about',
        }"
      >
        Über uns
      </NuxtLink>
      <NuxtLink
        to="/classrooms"
        v-if="loggedIn"
        class="block transition duration-300 hover:text-blue-600 dark:hover:text-blue-400"
        :class="{
          'font-bold text-blue-600 dark:text-blue-400':
            $route.path === '/classrooms',
        }"
      >
        Klassenräume
      </NuxtLink>
    </div>

    <!-- Main Content -->
    <main class="flex-grow">
      <NuxtPage />
    </main>

    <!-- Footer -->
    <footer
      class="bg-white px-8 py-4 text-center shadow-inner dark:bg-gray-800"
    >
      &copy; 2024 My Nuxt 3 Project. All rights reserved.
    </footer>
  </div>
</template>

<script setup>
import LoginIcon from "vue-material-design-icons/Login.vue";
import AccountIcon from "vue-material-design-icons/Account.vue";
import ThemeLightDark from "vue-material-design-icons/ThemeLightDark.vue";

const isDarkMode = ref(false);

const user = useSupabaseUser();
const loggedIn = ref(false);
const menuOpen = ref(false); // State for mobile menu visibility

watchEffect(() => {
  loggedIn.value = user.value !== null;
});

function toggleDarkMode() {
  isDarkMode.value = !isDarkMode.value;
  document.documentElement.classList.toggle("dark", isDarkMode.value);
}

// Toggle the mobile menu visibility
function toggleMenu() {
  menuOpen.value = !menuOpen.value;
}
</script>
