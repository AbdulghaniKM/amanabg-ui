<template>
  <div class="container mx-auto px-4 py-8">
    <div v-if="dataLoading" class="text-center">
      <p class="text-lg text-gray-600">Loading...</p>
    </div>
    <div v-else-if="dataError" class="text-center text-red-500">
      <p class="text-lg font-semibold">{{ dataError }}</p>
    </div>
    <div
      v-else-if="
        (hourlyData && hourlyData.length > 0) ||
        (minuteData && minuteData.length > 0)
      "
    >
      <div class="flex flex-col items-start gap-4 mb-8">
        <NuxtLink to="/data/pipes-water" class="text-DarkBlue hover:text-DarkBlue/80 transition-colors duration-300 flex items-center">
          <Icon name="mdi:arrow-left" class="mr-2" />
          Back to All Stations
        </NuxtLink>
        <h1 class="text-3xl font-bold text-gray-800 pb-2 w-full">
          Station Details
        </h1>
      </div>
      <div class="mb-6 rounded-lg bg-white p-6 shadow-lg">
        <p class="mb-2">
          <span class="pr-2">Name: </span>
          <span>{{ stationName || "N/A" }}</span>
        </p>
        <p class="mb-2">
          <span class="pr-2">City: </span>
          <span>{{ stationCity || "N/A" }}</span>
        </p>
        <div class="mr-4 flex items-center gap-4">
          <label for="from">From</label>
          <DatePicker
            v-model="fromDate"
            dateFormat="dd/mm/yy"
            class="h-10"
            @change="applyDateFilter"
          />
          <label for="to">To</label>
          <DatePicker
            v-model="toDate"
            dateFormat="dd/mm/yy"
            class="h-10"
            @change="applyDateFilter"
          />
          <button
            @click="resetDateFilter"
            class="rounded bg-DarkBlue px-4 py-2 font-bold text-white hover:bg-DarkBlue/90"
          >
            Reset
          </button>
        </div>
        <div class="mt-4 flex items-center gap-4">
          <label>Data Per Minute:</label>
          <DatePicker
            v-model="minuteDate"
            dateFormat="dd/mm/yy"
            class="h-10"
            :maxDate="new Date()"
            placeholder="Select Date"
          />
          <Button
            @click="fetchMinuteData"
            class="rounded !bg-DarkBlue px-4 py-2 font-bold !border-none !text-white hover:!bg-DarkBlue/90"
          >
            Submit
          </Button>
          <Button 
            @click="resetToHourlyData"
            class="rounded !bg-red-500 px-4 py-2 font-bold !border-none !text-white hover:!bg-red-600"
          >
            Reset
          </Button>
        </div>
        <p v-if="minuteDateError" class="mt-2 text-red-500">{{ minuteDateError }}</p>
      </div>
      <div class="rounded-lg bg-white p-6 shadow-lg">
        <h2 class="mb-4 text-2xl font-semibold text-gray-700">
          {{ dataType }} Data
        </h2>
        <Table
          :value="filteredData"
          :columns="columns"
          class="w-full"
          :sortField="'time'"
          :sortOrder="1"
        />
      </div>
    </div>
    <div v-else class="text-center">
      <p class="text-lg text-gray-600">No data available</p>
    </div>
    <div>
      <EChart :hourlyData="dataType === 'Hourly' ? hourlyData : minuteData" />
    </div>
  </div>
</template>

<script setup>
import { storeToRefs } from 'pinia';

const route = useRoute();
const stationDataHourStore = useStationDataHourStore();
const stationDataMinuteStore = useStationDataMinuteStore();

const {
  hourlyData,
  loading: hourLoading,
  error: hourError,
} = storeToRefs(stationDataHourStore);
const {
  minuteData,
  loading: minuteLoading,
  error: minuteError,
} = storeToRefs(stationDataMinuteStore);

// Helper function to get the start of day
const startOfDay = (date) => {
  const newDate = new Date(date);
  newDate.setHours(0, 0, 0, 0);
  return newDate;
};

// Helper function to get the end of day
const endOfDay = (date) => {
  const newDate = new Date(date);
  newDate.setHours(23, 59, 59, 999);
  return newDate;
};

// Helper function to subtract months from a date
const subtractMonths = (date, months) => {
  const newDate = new Date(date);
  newDate.setMonth(newDate.getMonth() - months);
  return newDate;
};

const fromDate = ref(startOfDay(subtractMonths(new Date(), 3)));
const toDate = ref(endOfDay(new Date()));
const minuteDate = ref(null);
const minuteDateError = ref('');
const { fetchHourlyData, fetchMinuteData: fetchMinuteDataStore } =
  stationDataHourStore;

const dataLoading = computed(() => hourLoading.value || minuteLoading.value);
const dataError = computed(() => hourError.value || minuteError.value);
const dataType = ref("Hourly");

const columns = computed(() => {
  const baseColumns = [
    { header: "Date", sortable: true, field: "date" },
    { header: "Time", sortable: true, field: "time" },
    {
      header: "Total Volume/Hour",
      sortable: true,
      field: "totalVolumePerHour",
    },
    { header: "Total Volume/Day", sortable: true, field: "totalVolumePerDay" },
    { header: "Press.", sortable: true, field: "pressure" },
    { header: "CL", sortable: true, field: "cl" },
    { header: "Turb.", sortable: true, field: "turbidity" },
    {
      header: "EC",
      sortable: true,
      field: "electricConductivity",
    },
  ];

  if (dataType.value === "Minute") {
    baseColumns.splice(2, 0, { header: "Q ( m3 / min )", sortable: true, field: "discharge" });
  }

  return baseColumns.map((column) => ({
    ...column,
    class:
      "!bg-DarkBlue sm:!text-sm !outline !outline-1 !outline-white !text-white font-semibold py-2",
  }));
});

const fetchData = async () => {
  const stationId = parseInt(route.params.id, 10);
  if (isNaN(stationId)) {
    console.error("Invalid station ID");
    return;
  }
  await stationDataHourStore.fetchHourlyData({ 
    stationId,
    fromDate: fromDate.value,
    toDate: toDate.value
  });
};

const fetchMinuteData = async () => {
  if (!minuteDate.value) {
    minuteDateError.value = 'Please select a date';
    return;
  }

  const selectedDate = new Date(minuteDate.value);
  const today = new Date();
  if (selectedDate > today) {
    minuteDateError.value = 'Selected date cannot be in the future';
    return;
  }

  const stationId = parseInt(route.params.id, 10);
  if (isNaN(stationId)) {
    console.error("Invalid station ID");
    return;
  }
  await stationDataMinuteStore.fetchMinuteData({
    stationId,
    date: minuteDate.value,
  });
  dataType.value = "Minute";
  minuteDateError.value = '';
};

const applyDateFilter = () => {
  if (fromDate.value && toDate.value) {
    fromDate.value.setHours(0, 0, 0, 0);
    toDate.value.setHours(23, 59, 59, 999);
  }
};

const resetDateFilter = () => {
  fromDate.value = startOfDay(subtractMonths(new Date(), 3));
  toDate.value = endOfDay(new Date());
  fetchData(); // Refetch data with reset dates
};

const resetToHourlyData = () => {
  dataType.value = "Hourly";
  minuteDate.value = null;
  minuteDateError.value = '';
};

const formattedData = computed(() => {
  const data =
    dataType.value === "Hourly" ? hourlyData.value : minuteData.value;
  if (!data) return [];

  return data.map((item) => {
    const date = new Date(item.timeStamp);
    return {
      ...item,
      date: date.toLocaleDateString('en-GB', { day: '2-digit', month: '2-digit', year: 'numeric' }),
      time: date.toLocaleTimeString('en-US', { hour: '2-digit', minute: '2-digit' }),
    };
  });
});

const filteredData = computed(() => {
  if (!fromDate.value || !toDate.value) return formattedData.value;

  return formattedData.value.filter((item) => {
    const itemDate = new Date(item.timeStamp);
    return itemDate >= fromDate.value && itemDate <= toDate.value;
  });
});

const stationName = ref("N/A");
const stationCity = ref("N/A");

onMounted(() => {
  fetchData();
  if (process.client) {
    stationName.value = window.localStorage.getItem("stationName") || "N/A";
    stationCity.value = window.localStorage.getItem("stationCity") || "N/A";
  }
});

watch([fromDate, toDate], applyDateFilter);
</script>

<style>
.p-datepicker-input {
  @apply !bg-gray-200 !text-black;
}
</style>
