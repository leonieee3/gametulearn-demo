<template>
  <div class="space-y-6 p-6">
    <h2 class="text-2xl font-bold text-gray-900 dark:text-white">
      Insights für {{ student_name }}
    </h2>
    <!-- chart type selection -->
    <div class="flex items-center space-x-4">
      <select
        v-model="selectedChart"
        class="rounded-lg border bg-gray-100 p-2 focus:outline-none focus:ring-2 focus:ring-blue-500 dark:border-gray-600 dark:bg-gray-700"
      >
        <option value="generalData">Allgemeine Daten</option>
        <option value="weeklyPlaytime">wöchentliche Spielzeit</option>
      </select>
    </div>

    <!-- Date Range Picker for weekly playtime -->
    <div v-if="selectedChart === 'weeklyPlaytime'" class="relative">
      <VueDatePicker
        v-model="dateRange"
        week-picker
        :enable-time-picker="false"
        :locale="'de'"
        :select-text="'Auswählen'"
        :cancel-text="'Abbrechen'"
        :format="'dd.MM.yyyy'"
        @update:modelValue="updateDateRange"
        @cleared="resetDateRange"
        class="absolute left-0 top-full z-10 rounded border bg-white shadow dark:border-gray-600 dark:bg-gray-700"
      />
    </div>

    <!-- Display general data -->
    <div
      v-if="!isLoading && selectedChart === 'generalData'"
      class="grid grid-cols-1 gap-6 md:grid-cols-2"
    >
      <div
        class="rounded-lg bg-gradient-to-r from-blue-400 to-blue-600 p-6 text-center shadow-md"
      >
        <h3 class="text-lg font-semibold text-white">Gesamtspielzeit</h3>
        <p class="text-2xl font-bold text-white">
          {{ formatPlaytime(generalData.totalPlaytime) }}
        </p>
      </div>

      <div
        class="rounded-lg bg-gradient-to-r from-green-400 to-green-600 p-6 text-center shadow-md"
      >
        <h3 class="text-lg font-semibold text-white">
          Spielzeit in den letzten 7 Tagen
        </h3>
        <p class="text-2xl font-bold text-white">
          {{ formatPlaytime(generalData.weeklyPlaytime) }}
        </p>
      </div>

      <div
        class="rounded-lg bg-gradient-to-r from-pink-400 to-pink-600 p-6 text-center shadow-md"
      >
        <h3 class="text-lg font-semibold text-white">Level</h3>
        <p class="text-2xl font-bold text-white">{{ generalData.level }}</p>
      </div>

      <div
        class="rounded-lg bg-gradient-to-r from-yellow-400 to-orange-500 p-6 text-center shadow-md"
      >
        <h3 class="text-lg font-semibold text-white">Punktzahl</h3>
        <p class="text-2xl font-bold text-white">{{ generalData.score }}</p>
      </div>
    </div>

    <!-- Plotly chart -->
    <client-only v-else-if="!isLoading">
      <nuxt-plotly
        :data="chartData"
        :layout="layout"
        :config="config"
        style="width: 100%"
        @on-ready="myChartOnReady"
      ></nuxt-plotly>
    </client-only>
  </div>
</template>

<script setup lang="ts">
import VueDatePicker from "@vuepic/vue-datepicker";
import "@vuepic/vue-datepicker/dist/main.css";
import { formatPlaytime, convertHHMMSSToMinutes } from "~/utils/insights";
import { blueShades } from "~/utils/colors";
import type {
  NuxtPlotlyConfig,
  NuxtPlotlyData,
  NuxtPlotlyLayout,
  NuxtPlotlyHTMLElement,
} from "nuxt-plotly";

const props = defineProps<{
  student_id: string;
  student_name: string;
  game_id: string;
  classroom_id: string;
}>();

const supabase = useSupabaseClient();
const isLoading = ref(true);
const selectedChart = ref("generalData");
const chartData = ref<NuxtPlotlyData>([]);
const generalData = ref({
  totalPlaytime: "00:00:00",
  weeklyPlaytime: "00:00:00",
  score: 0,
  level: 0,
});

const dateRange = ref();
const startDate = ref();
const endDate = ref();

// Update date range when changed
const updateDateRange = (range: [Date, Date] | null) => {
  if (range && range.length === 2) {
    startDate.value = `${range[0].getFullYear()}-${(range[0].getMonth() + 1).toString().padStart(2, "0")}-${range[0].getDate().toString().padStart(2, "0")}`;
    endDate.value = `${range[1].getFullYear()}-${(range[1].getMonth() + 1).toString().padStart(2, "0")}-${range[1].getDate().toString().padStart(2, "0")}`;

    fetchWeeklyPlaytime();
  }
};

// Reset date range to the current week (Monday to Sunday)
const resetDateRange = () => {
  const now = new Date();
  const dayOfWeek = now.getDay(); // 0: Sunday, 1: Monday, ..., 6: Saturday
  const diffToMonday = dayOfWeek === 0 ? 6 : dayOfWeek - 1; // If today is Sunday (0), go back to Saturday (6)

  const startOfWeek = new Date(now);
  startOfWeek.setDate(now.getDate() - diffToMonday); // Set to Monday

  // Setze das Enddatum der Woche (Sonntag)
  const endOfWeek = new Date(startOfWeek);
  endOfWeek.setDate(startOfWeek.getDate() + 6); // Sunday of the current week

  // Format the date to ISO format (YYYY-MM-DD)
  startDate.value = startOfWeek.toISOString().split("T")[0];
  endDate.value = endOfWeek.toISOString().split("T")[0];

  dateRange.value = [startOfWeek, endOfWeek];
  fetchWeeklyPlaytime();
};

// Layout configuration for the chart
const layout = computed(() => {
  return selectedChart.value === "weeklyPlaytime"
    ? {
        title: "wöchentliche Spielzeit",
        xaxis: { fixedrange: true },
        yaxis: { title: "Spielzeit (min)", fixedrange: true },
      }
    : {};
});

// Config for Plotly chart
const config: NuxtPlotlyConfig = { scrollZoom: false, displayModeBar: false };

// German days mapping for the chart (1 = Monday, 7 = Sunday)
const germanDays = {
  1: "Montag",
  2: "Dienstag",
  3: "Mittwoch",
  4: "Donnerstag",
  5: "Freitag",
  6: "Samstag",
  7: "Sonntag",
};

// Fetch data for chart and general information
const fetchChartData = async () => {
  isLoading.value = true;
  if (selectedChart.value !== "weeklyPlaytime") {
    await fetchGeneralData();
  }

  isLoading.value = false;
};

// Fetch weekly playtime data for the selected date range
const fetchWeeklyPlaytime = async () => {
  try {
    const weeklyPlaytimes: {
      day_number: number;
      day: string;
      playtime: number;
    }[] = [];

    const { data, error } = await supabase.rpc("get_weekly_playtime_by_day", {
      p_user_id: props.student_id,
      p_game_id: props.game_id,
      p_classroom_id: props.classroom_id,
      p_start_date: startDate.value + " 00:00:00",
      p_end_date: endDate.value + " 23:59:59",
    } as any);

    if (error || data === null) {
      console.error("Error fetching weekly playtime: ", error);
      return;
    }

    // Process and convert the playtime to minutes
    (data as { day_number: number; total_playtime: string }[]).forEach(
      (dayData) => {
        const dayOfWeek =
          germanDays[dayData.day_number as keyof typeof germanDays] ||
          "Unbekannt";
        const totalPlaytimeMinutes = convertHHMMSSToMinutes(
          dayData.total_playtime,
        );

        weeklyPlaytimes.push({
          day_number: dayData.day_number,
          day: dayOfWeek,
          playtime: totalPlaytimeMinutes,
        });
      },
    );

    // Sort by day_number to ensure Monday comes first
    weeklyPlaytimes.sort((a, b) => a.day_number - b.day_number);

    // Extract days and playtime for chart
    const daysOfWeek = weeklyPlaytimes.map((entry) => entry.day);
    const playtimeValues = weeklyPlaytimes.map((entry) => entry.playtime);

    const colors = playtimeValues.map((_, index) => blueShades[index % 3]);

    // Set chart data
    chartData.value = [
      {
        x: daysOfWeek,
        y: playtimeValues,
        type: "bar",
        marker: {
          color: colors,
          line: {
            color: "rgb(0, 0, 0)",
            width: 1,
          },
        },
      },
    ];
  } catch (error) {
    console.error("Error creating weekly playtime chart data:", error);
  }
};

// Fetch general data (total playtime, weekly playtime, score, level)
const fetchGeneralData = async () => {
  isLoading.value = true;

  try {
    // Get total playtime
    const { data: totalPlaytime, error: totalPlaytimeError } =
      await supabase.rpc("get_total_playtime", {
        p_user_id: props.student_id,
        p_game_id: props.game_id,
        p_classroom_id: props.classroom_id,
      } as any);
    if (totalPlaytimeError) {
      console.error("Error fetching total playtime:", totalPlaytimeError);
    }

    // Get playtime for the last 7 days
    const { data: weeklyPlaytime, error: weeklyPlaytimeError } =
      await supabase.rpc("get_weekly_playtime", {
        p_user_id: props.student_id,
        p_game_id: props.game_id,
        p_classroom_id: props.classroom_id,
      } as any);

    if (weeklyPlaytimeError) {
      console.error("Error fetching weekly playtime:", weeklyPlaytimeError);
    }

    // Get score
    const { data: score, error: scoreError } = await supabase.rpc("get_score", {
      p_user_id: props.student_id,
      p_game_id: props.game_id,
      p_classroom_id: props.classroom_id,
    } as any);

    if (scoreError) {
      console.error("Error fetching score:", scoreError);
    }

    // Get level
    const { data: level, error: levelError } = await supabase.rpc("get_level", {
      p_user_id: props.student_id,
      p_game_id: props.game_id,
      p_classroom_id: props.classroom_id,
    } as any);

    if (levelError) {
      console.error("Error fetching level:", levelError);
    }
    generalData.value = {
      totalPlaytime: totalPlaytime || "00:00:00",
      weeklyPlaytime: weeklyPlaytime || "00:00:00",
      score: score || 0,
      level: level || 0,
    };
  } catch (error) {
    console.error("Error fetching general data:", error);
  }

  isLoading.value = false;
};

// Event handler for Plotly chart ready event
function myChartOnReady(plotlyHTMLElement: NuxtPlotlyHTMLElement) {
  const { $plotly } = useNuxtApp();

  plotlyHTMLElement.on?.("plotly_afterplot", function () {
    console.log("done plotting");
  });
}

onMounted(async () => {
  fetchChartData();
});

// Re-fetch data when chart selection changes
watch(selectedChart, fetchChartData);
watch(selectedChart, () => {
  if (selectedChart.value === "weeklyPlaytime") {
    resetDateRange();
  }
});

// Re-fetch data when student changes
watch(
  () => [props.student_id, props.student_name],
  () => {
    selectedChart.value = "generalData"; // Reset to general data
    fetchChartData(); // Fetch new data if student changes
  },
);
</script>
