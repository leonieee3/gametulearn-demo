<template>
  <div v-if="classroom_id" class="space-y-6 p-6">
    <h2 class="text-2xl font-bold text-gray-900 dark:text-white">
      Insights für {{ classroom_name }}
    </h2>

    <!-- Chart type selection -->
    <div class="flex items-center space-x-4">
      <select
        v-model="selectedChart"
        class="rounded-lg border bg-gray-100 p-2 focus:outline-none focus:ring-2 focus:ring-blue-500 dark:border-gray-600 dark:bg-gray-700"
      >
        <option value="generalData">Allgemeine Daten</option>
        <option value="levelDistribution">Level-Verteilung</option>
        <option value="scoreDistribution">Punktzahl-Verteilung</option>
        <option value="playtimeDistribution">Spielzeit-Verteilung</option>
      </select>
    </div>

    <!-- Date range picker for playtime distribution -->
    <div v-if="selectedChart === 'playtimeDistribution'" class="relative">
      <VueDatePicker
        v-model="dateRange"
        range
        :enable-time-picker="false"
        :locale="'de'"
        :select-text="'Auswählen'"
        :cancel-text="'Abbrechen'"
        :format="'dd.MM.yyyy'"
        @update:modelValue="updateDateRange"
        @cleared="resetDateRange"
        class="absolute left-0 top-full z-10 rounded-lg border bg-white shadow-lg dark:border-gray-600 dark:bg-gray-700"
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
        <h3 class="text-lg font-semibold text-white">
          Durchschnittliche Spielzeit in den letzten 7 Tagen
        </h3>
        <p class="text-2xl font-bold text-white">
          {{ formatPlaytime(generalData.avgPlaytime) }}
        </p>
      </div>

      <div
        class="rounded-lg bg-gradient-to-r from-green-400 to-green-600 p-6 text-center shadow-md"
      >
        <h3 class="text-lg font-semibold text-white">Aktive Schüler</h3>
        <p class="text-2xl font-bold text-white">
          {{ generalData.activeStudentsCount }} von {{ studentIds.length }}
          <span v-if="studentIds.length > 0">
            ({{
              Math.round(
                (generalData.activeStudentsCount / studentIds.length) * 100,
              )
            }}%)
          </span>
        </p>
      </div>

      <div
        class="rounded-lg bg-gradient-to-r from-pink-400 to-pink-600 p-6 text-center shadow-md"
      >
        <h3 class="text-lg font-semibold text-white">
          Durchschnittliches Level
        </h3>
        <p class="text-2xl font-bold text-white">
          {{ Math.round(generalData.avgLevel) }}
        </p>
      </div>

      <div
        class="rounded-lg bg-gradient-to-r from-yellow-400 to-orange-500 p-6 text-center shadow-md"
      >
        <h3 class="text-lg font-semibold text-white">
          Durchschnittliche Punktzahl
        </h3>
        <p class="text-2xl font-bold text-white">
          {{ Math.round(generalData.avgScore) }}
        </p>
      </div>
    </div>

    <!-- Plotly chart -->
    <client-only v-else-if="!isLoading">
      <nuxt-plotly
        :data="chartData"
        :layout="layout"
        :config="config"
        style="width: 100%; height: 400px"
        class="bg-white p-6 dark:bg-gray-700"
      ></nuxt-plotly>
    </client-only>
  </div>

  <!-- Display message when no classroom is selected -->
  <div v-else class="mt-6 text-center text-gray-500 dark:text-gray-400">
    <p>Bitte wählen Sie eine Klasse aus, um die Insights anzuzeigen.</p>
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
  game_id: string;
  classroom_id: string;
  classroom_name: string;
}>();

const supabase = useSupabaseClient();
const studentIds = ref<string[]>([]);
const isLoading = ref(true);
const selectedChart = ref("generalData");
const chartData = ref<NuxtPlotlyData>([]);
const generalData = ref({
  avgPlaytime: "00:00:00",
  avgScore: 0,
  avgLevel: 0,
  activeStudentsCount: 0,
});

const dateRange = ref();
const startDate = ref();
const endDate = ref();

// Update date range when changed
const updateDateRange = (range: [Date, Date] | null) => {
  if (range && range.length === 2) {
    startDate.value = `${range[0].getFullYear()}-${(range[0].getMonth() + 1).toString().padStart(2, "0")}-${range[0].getDate().toString().padStart(2, "0")}`;
    endDate.value = `${range[1].getFullYear()}-${(range[1].getMonth() + 1).toString().padStart(2, "0")}-${range[1].getDate().toString().padStart(2, "0")}`;

    fetchAveragePlaytimeDistribution();
  }
};

// Reset date range to the past 7 days
const resetDateRange = () => {
  startDate.value = new Date(new Date().setDate(new Date().getDate() - 7))
    .toISOString()
    .split("T")[0];
  endDate.value = new Date().toISOString().split("T")[0];
  dateRange.value = [startDate.value, endDate.value];
  fetchAveragePlaytimeDistribution();
};

// Config for Plotly chart
const config: NuxtPlotlyConfig = { scrollZoom: false, displayModeBar: false };

// Layout configuration based on the selected chart type
const layout = computed(() => {
  return selectedChart.value === "levelDistribution"
    ? {
        title: "Verteilung der Level",
        xaxis: {
          title: "Level",
          tickmode: "linear",
          tick0: 1,
          dtick: 1,
          fixedrange: true,
        },
        yaxis: {
          title: "Anzahl der Schüler",
          tickmode: "linear",
          tick0: 0,
          dtick: 1,
          fixedrange: true,
        },
      }
    : selectedChart.value === "scoreDistribution"
      ? {
          title: "Verteilung der Punktzahlen",
          xaxis: { title: "Punktzahl", fixedrange: true },
          yaxis: { title: "Anzahl der Schüler", fixedrange: true },
        }
      : selectedChart.value === "playtimeDistribution"
        ? {
            title: "Verteilung der durchschnittlichen Spielzeit",
            xaxis: { title: "Spielzeit (min)", fixedrange: true },
            yaxis: { title: "Anzahl der Schüler", fixedrange: true },
          }
        : {};
});

// Fetch all students in the classroom
const fetchStudents = async () => {
  const { data, error } = await supabase.rpc("get_students_for_classroom", {
    p_classroom_id: props.classroom_id,
  } as any);

  if (error) {
    console.error("Error fetching the students:", error);
    return;
  }
  studentIds.value = (data as any[]).map((item: any) => item.student_id);
};

// Fetch chart data based on the selected chart type
const fetchChartData = async () => {
  isLoading.value = true;

  if (selectedChart.value === "levelDistribution") {
    await fetchLevelDistribution();
  } else if (selectedChart.value === "scoreDistribution") {
    await fetchScoreDistribution();
  } else if (selectedChart.value === "playtimeDistribution") {
    // await fetchAveragePlaytimeDistribution();
  } else {
    await fetchGeneralData();
  }

  isLoading.value = false;
};

// Fetch data for score distribution
const fetchScoreDistribution = async () => {
  try {
    const { data, error } = await supabase.rpc("get_scores_for_students", {
      p_game_id: props.game_id,
      p_classroom_id: props.classroom_id,
      p_student_ids: studentIds.value,
    } as any);
    if (error) {
      console.error("Error fetching scores:", error);
      return;
    }

    const scores = (data as any[]).map((item: { score: number }) => item.score);
    const colors = scores.map((_: any, index: any) => blueShades[index % 3]);

    // Plotly will automatically group scores into 1000 intervals
    chartData.value = [
      {
        x: scores,
        type: "histogram",
        xbins: {
          start: 0,
          size: 1000,
        },
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
    console.error("Error creating score distribution chart data:", error);
  }
};

// Fetch data for level distribution
const fetchLevelDistribution = async () => {
  try {
    const { data, error } = await supabase.rpc("get_levels_for_students", {
      p_game_id: props.game_id,
      p_classroom_id: props.classroom_id,
      p_student_ids: studentIds.value,
    } as any);

    if (error) {
      console.error("Error fetching the levels:", error);
      return;
    }

    const levelCount: Record<number, number> = {};
    (data as any[]).forEach((item: { level: number }) => {
      levelCount[item.level] = (levelCount[item.level] || 0) + 1;
    });

    const levels = Object.keys(levelCount).map(Number);
    const studentCountAtEachLevel = levels.map((level) => levelCount[level]);
    const colors = levels.map((_, index) => blueShades[index % 3]);
    chartData.value = [
      {
        x: levels,
        y: studentCountAtEachLevel,
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
    console.error("Error at creating chart data:", error);
  }
};

// Fetch average playtime distribution for all students within the date range
const fetchAveragePlaytimeDistribution = async () => {
  try {
    const averagePlaytimes = [];

    for (const studentId of studentIds.value) {
      const { data, error } = await supabase.rpc(
        "calculate_playtime_in_range",
        {
          p_user_id: studentId,
          p_game_id: props.game_id,
          p_classroom_id: props.classroom_id,
          p_start_date: startDate.value + " 00:00:00",
          p_end_date: endDate.value + " 23:59:59",
        } as any,
      );

      if (error) {
        console.error(
          `Error fetching total playtime for user ${studentId}:`,
          error,
        );
        continue;
      }

      const totalPlaytimeMinutes = convertHHMMSSToMinutes(data);
      averagePlaytimes.push(totalPlaytimeMinutes);
    }

    chartData.value = [
      {
        x: averagePlaytimes,
        type: "histogram",
        xbins: {
          start: 0,
          size: 10, // bin size of 10 minutes
        },
        marker: {
          bgcolor: blueShades,
          line: {
            color: "rgb(0, 0, 0)",
            width: 1.5,
          },
        },
      },
    ];
  } catch (error) {
    console.error("Error creating playtime distribution chart data:", error);
  }
};

// Fetch general data about the classroom (avg score, avg playtime, etc.)
const fetchGeneralData = async () => {
  isLoading.value = true;

  try {
    // Active student count
    const { data: activeStudentsCount, error: activeStudentsError } =
      await supabase.rpc("count_active_students", {
        p_classroom_id: props.classroom_id,
        p_game_id: props.game_id,
      } as any);

    if (activeStudentsError) {
      console.error(
        "Error fetching active students count:",
        activeStudentsError,
      );
    }

    // Average playtime in the last 7 days
    const { data: avgPlaytime, error: playtimeError } = await supabase.rpc(
      "calculate_average_weekly_playtime",
      {
        p_classroom_id: props.classroom_id,
        p_game_id: props.game_id,
      } as any,
    );

    if (playtimeError) {
      console.error("Error fetching average weekly playtime:", playtimeError);
    }

    // Average score
    const { data: avgScore, error: scoreError } = await supabase.rpc(
      "calculate_average_score",
      {
        p_classroom_id: props.classroom_id,
        p_game_id: props.game_id,
      } as any,
    );

    if (scoreError) {
      console.error("Error fetching average score:", scoreError);
    }

    // Average level
    const { data: avgLevel, error: levelError } = await supabase.rpc(
      "calculate_average_level",
      {
        p_classroom_id: props.classroom_id,
        p_game_id: props.game_id,
      } as any,
    );

    if (levelError) {
      console.error("Error fetching average level:", levelError);
    }
    generalData.value = {
      avgPlaytime: avgPlaytime || "00:00:00",
      avgScore: avgScore || 0,
      avgLevel: avgLevel || 0,
      activeStudentsCount: activeStudentsCount || 0,
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
  await fetchStudents();
  fetchChartData();
});

// Watch for changes in chart selection and update accordingly
watch(selectedChart, fetchChartData);
watch(selectedChart, () => {
  if (selectedChart.value === "playtimeDistribution") {
    resetDateRange();
  }
});
</script>
