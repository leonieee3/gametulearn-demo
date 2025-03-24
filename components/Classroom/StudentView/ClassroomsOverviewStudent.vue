<template>
  <div class="mx-auto max-w-4xl px-4 py-8">
    <!-- Header with the option to join a classroom -->
    <div class="mb-6 flex items-center justify-between">
      <h1 class="text-2xl font-bold text-gray-900 dark:text-gray-100">
        Deine Klassenräume
      </h1>
      <!-- Form to join with 6 code input fields and the join button -->
      <form @submit.prevent="joinClassroom" class="flex items-center space-x-2">
        <!-- Access code form with 6 fields -->
        <div class="flex space-x-1">
          <input
            v-for="(digit, index) in codeDigits"
            :key="index"
            v-model="codeDigits[index]"
            type="text"
            maxlength="1"
            class="code-input h-10 w-10 rounded-lg border border-gray-300 text-center text-xl focus:outline-none focus:ring-2 focus:ring-blue-500 dark:border-gray-600 dark:bg-gray-700 dark:text-gray-100 dark:focus:ring-blue-400"
            @input="moveFocus(index)"
            @focus="clearError"
            placeholder="X"
          />
        </div>
        <button
          type="submit"
          class="rounded-lg bg-blue-500 px-3 py-2 text-white hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-blue-300 dark:hover:bg-blue-600 dark:focus:ring-blue-500"
          :disabled="!isCodeValid"
        >
          Beitreten
        </button>
      </form>
    </div>

    <!-- Error message if the access code is invalid -->
    <div v-if="invalidError" class="mb-4 mt-4 text-red-500">
      <p>
        {{ errorMessage }}
      </p>
    </div>

    <!-- User's classrooms -->
    <div v-if="classrooms.length === 0" class="py-8 text-center text-gray-600">
      <p>Du bist noch keinem Klassenraum beigetreten.</p>
    </div>

    <div v-else>
      <ul class="space-y-4">
        <li
          v-for="classroom in classrooms"
          :key="classroom.id"
          class="flex items-center justify-between rounded-lg bg-white p-4 shadow transition-shadow hover:shadow-lg dark:bg-gray-800"
        >
          <div>
            <router-link
              :to="`/classrooms/${classroom.access_code}`"
              class="text-lg font-semibold text-gray-800 hover:underline dark:text-gray-100"
            >
              {{ classroom.name }}
            </router-link>
            <p class="text-sm text-gray-600 dark:text-gray-400">
              {{ classroom.description }}
            </p>
          </div>
          <div class="flex items-center space-x-2">
            <!-- Button to leave the classroom -->
            <button
              @click="leaveClassroom(classroom.id)"
              class="rounded-lg px-3 py-2 text-black transition hover:bg-gray-400 dark:text-gray-300"
            >
              <DeleteIcon class="scale-150 transform transition-transform" />
            </button>
          </div>
        </li>
      </ul>
    </div>
  </div>
</template>

<script setup>
import DeleteIcon from "vue-material-design-icons/Delete.vue";

const codeDigits = ref(["", "", "", "", "", ""]);
const invalidError = ref(false);
const errorMessage = ref("");
const classrooms = ref([]);
const supabase = useSupabaseClient();
const user = useSupabaseUser();
const accessCode = ref("");
const router = useRouter();

// Ensure the access code is 6 characters long before sending the request
const isCodeValid = computed(() => codeDigits.value.join("").length === 6);

// Fetch the classrooms the user is part of
async function fetchClassrooms() {
  try {
    const { data, error: studentError } = await supabase
      .from("student_classroom")
      .select("classroom_id")
      .eq("student_id", user.value.id);

    if (studentError) throw studentError;

    const classroomIds = data.map((entry) => entry.classroom_id);

    const { data: classroomsData, error: classroomError } = await supabase
      .from("classrooms")
      .select("*")
      .in("id", classroomIds);

    if (classroomError) throw classroomError;

    classrooms.value = classroomsData;
  } catch (error) {
    console.error("Error fetching classrooms:", error);
  }
}

// Join a classroom with the access code
async function joinClassroom() {
  if (!isCodeValid.value) {
    return; // Prevents submission if the code is invalid
  }

  try {
    accessCode.value = codeDigits.value.join("");

    const { data: classroom, error: fetchError } = await supabase.rpc(
      "get_classroom_by_access_code",
      {
        access_code_input: accessCode.value,
      },
    );

    if (fetchError || (classroom && Object.keys(classroom).length === 0)) {
      invalidError.value = true;
      errorMessage.value =
        "Der Zugangscode konnte nicht gefunden werden. Bitte prüfe den Code.";
      codeDigits.value = ["", "", "", "", "", ""];
      return;
    }

    const { error: insertError } = await supabase
      .from("student_classroom")
      .upsert(
        [
          {
            student_id: user.value.id,
            classroom_id: classroom[0].id,
          },
        ],
        { onConflict: ["student_id", "classroom_id"] },
      );

    if (insertError) throw insertError;

    router.push(`/classrooms/${classroom[0].access_code}`);
  } catch (error) {
    console.error("Error joining classroom:", error);
  }
}

// Leave a classroom
async function leaveClassroom(classroomId) {
  try {
    const { error } = await supabase
      .from("student_classroom")
      .delete()
      .eq("student_id", user.value.id)
      .eq("classroom_id", classroomId);

    if (error) throw error;

    classrooms.value = classrooms.value.filter(
      (classroom) => classroom.id !== classroomId,
    );
  } catch (error) {
    console.error("Error leaving classroom:", error);
  }
}

// Move focus to the next input field when a user enters a digit
const moveFocus = (index) => {
  if (codeDigits.value[index] && index < 5) {
    const nextInput = document.querySelectorAll(".code-input")[index + 1];
    nextInput?.focus();
  }
};

// Clear the error message when the user starts typing again
const clearError = () => {
  invalidError.value = false;
  errorMessage.value = "";
};

// Fetch classrooms when the page loads
onMounted(fetchClassrooms);
</script>
