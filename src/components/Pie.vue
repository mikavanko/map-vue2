<script>
import { reactive, computed, ref, onMounted } from "vue";
import { Pie } from "vue-chartjs/legacy";
import ChartDataLabels from "chartjs-plugin-datalabels";

import {
  Chart as ChartJS,
  Title,
  Tooltip,
  Legend,
  ArcElement,
  CategoryScale,
} from "chart.js";

ChartJS.register(
  Title,
  Tooltip,
  Legend,
  ArcElement,
  CategoryScale,
  ChartDataLabels
);

export default {
  components: {
    Pie,
  },
  emits: ["click-section"],
  props: {
    chartId: {
      type: String,
      default: "pie-chart",
    },
    datasetIdKey: {
      type: String,
      default: "label",
    },
    width: {
      type: Number,
      default: 400,
    },
    height: {
      type: Number,
      default: 400,
    },
    cssClasses: {
      default: "",
      type: String,
    },
    styles: {
      type: Object,
      default: () => {},
    },
    plugins: {
      type: Array,
      default: () => [],
    },
    features: {
      type: Array,
      default: () => [],
    },
    regionsCount: {
      type: Object,
      default: () => {},
    },
  },
  setup(props, { emit }) {
    const pieRef = ref();

    const labels = computed(() =>
      props.features.map((el) => el.properties.description)
    );
    const datasetsBackgroundColor = computed(() =>
      props.features.map((el) => el.properties.fill)
    );
    const datasetsData = computed(() =>
      props.features.map((el) => props.regionsCount[el.id])
    );
    const state = reactive({
      chartData: {
        labels,
        datasets: [
          {
            backgroundColor: datasetsBackgroundColor,
            data: datasetsData,
          },
        ],
      },
      chartOptions: {
        responsive: true,
        maintainAspectRatio: false,
        plugins: {
          datalabels: {
            display: true,
            color: "#000",
            backgroundColor: "#fff",
            borderRadius: 3,
            font: {
              size: 14,
            },
          },
        },
        onClick: function (e, elements) {
          var idx = elements[0].index;
          const curFeature = props.features[idx];

          emit("click-section", curFeature);
        },
      },
    });

    return {
      state,
      pieRef,
    };
  },
};
</script>

<template>
  <Pie
    ref="pieRef"
    class="pie-chart"
    :chart-options="state.chartOptions"
    :chart-data="state.chartData"
    :chart-id="chartId"
    :dataset-id-key="datasetIdKey"
    :plugins="plugins"
    :css-classes="cssClasses"
    :styles="styles"
    :width="width"
    :height="height"
  />
</template>

<style lang="scss" scoped>
@media all and (max-width: 768px) {
  :deep() {
    #pie-chart {
      height: auto !important;
      width: 100% !important;
    }
  }
}
</style>