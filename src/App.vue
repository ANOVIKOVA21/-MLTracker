<script setup lang="ts">
import Message from 'primevue/message'
import FileUpload from 'primevue/fileupload'
import Checkbox from 'primevue/checkbox'
import Chart from 'primevue/chart'
import Divider from 'primevue/divider'
import ScrollPanel from 'primevue/scrollpanel'
import Papa from 'papaparse'
import 'primeicons/primeicons.css'
import type { FileUploadSelectEvent } from 'primevue/fileupload'
import { ref, computed, onMounted } from 'vue'

type ExperimentRaw = {
  experiment_id: string
  metric_name: string
  step: string
  value: string
}

type Experiment = {
  experiment_id: string
  metrics: Map<string, ExperimentMetric>
}

type ExperimentMetric = {
  metric_name: string
  steps: ExperimentMetricStep[]
}

type ExperimentMetricStep = {
  step: number
  value: number
}

const fileName = ref('')
const dataReady = ref(false)
const experiments = ref<Map<string, Experiment>>(new Map<string, Experiment>())
const selectedExperiments = ref<string[]>([])
const chartOptions = ref()
const uniqueMetrics = computed(() => {
  const metricNames = new Set<string>()

  selectedExperiments.value.forEach((id) => {
    const experiment = experiments.value.get(id)
    if (experiment) {
      experiment.metrics.forEach((_, metric_name) => {
        metricNames.add(metric_name)
      })
    }
  })

  return Array.from(metricNames)
})

function onFileSelect(event: FileUploadSelectEvent) {
  selectedExperiments.value = []
  dataReady.value = false
  const file = event.files[0]
  fileName.value = file.name
  const reader = new FileReader()

  reader.onload = async (e) => {
    if (!e.target) return
    const csv = e.target.result as string
    const result = Papa.parse(csv, { header: true, skipEmptyLines: true })
    const experimentsRaw = result.data as ExperimentRaw[]
    experiments.value = groupExperiments(experimentsRaw)
    dataReady.value = true
  }
  reader.readAsText(file)
}

function groupExperiments(rawList: ExperimentRaw[]): Map<string, Experiment> {
  const experimentMap = new Map<string, Experiment>()

  for (const raw of rawList) {
    if (!experimentMap.has(raw.experiment_id)) {
      experimentMap.set(raw.experiment_id, {
        experiment_id: raw.experiment_id,
        metrics: new Map<string, ExperimentMetric>(),
      })
    }

    const experiment = experimentMap.get(raw.experiment_id)!

    if (!experiment.metrics.has(raw.metric_name)) {
      experiment.metrics.set(raw.metric_name, {
        metric_name: raw.metric_name,
        steps: [],
      })
    }

    const metric = experiment.metrics.get(raw.metric_name)!
    metric.steps.push({
      step: Number(raw.step),
      value: Number(raw.value),
    })
  }

  for (const experiment of experimentMap.values()) {
    for (const metric of experiment.metrics.values()) {
      metric.steps.sort((a, b) => a.step - b.step)
    }
  }

  return experimentMap
}

onMounted(() => {
  chartOptions.value = setChartOptions()
})

function generateColor(index: number) {
  const hue = (index * 137.508) % 360
  return `hsl(${hue}, 65%, 55%)`
}

function getChartData(metricName: string) {
  const labelsX: string[] = []
  const datasets: object[] = []

  selectedExperiments.value.forEach((expId, i) => {
    const experiment = experiments.value.get(expId)
    const metric = experiment?.metrics.get(metricName)
    const steps = metric?.steps || []
    const stepValues: number[] = []

    for (const step of steps) {
      stepValues.push(step.value)
    }

    if (labelsX.length === 0) {
      for (const step of steps) {
        labelsX.push(step.step.toString())
      }
    }

    datasets.push({
      label: expId,
      data: stepValues,
      fill: false,
      borderColor: generateColor(i),
      borderWidth: 2,
      tension: 0.4,
      pointRadius: 0,
    })
  })

  return { labels: labelsX, datasets }
}
const setChartOptions = () => {
  const documentStyle = getComputedStyle(document.documentElement)
  const textColor = documentStyle.getPropertyValue('--p-text-color')
  const textColorSecondary = documentStyle.getPropertyValue('--p-text-muted-color')
  const surfaceBorder = documentStyle.getPropertyValue('--p-content-border-color')
  return {
    maintainAspectRatio: false,
    aspectRatio: 0.6,
    plugins: {
      legend: {
        labels: {
          color: textColor,
        },
      },
    },
    scales: {
      x: {
        type: 'linear',
        ticks: {
          stepSize: 1000,
          color: textColorSecondary,
        },
        grid: {
          color: surfaceBorder,
        },
      },
      y: {
        ticks: {
          color: textColorSecondary,
        },
        grid: {
          color: surfaceBorder,
        },
      },
    },
  }
}
</script>

<template>
  <aside>
    <div class="wrapper">
      <h1>Experiment tracking</h1>
      <Message severity="info" icon="pi pi-info-circle" closable
        >Click to upload your csv file</Message
      >
      <div class="input">
        <FileUpload
          mode="basic"
          @select="onFileSelect"
          customUpload
          auto
          severity="primary"
          chooseLabel="Upload"
          class="p-button-outlined"
        />
        <p v-if="fileName" class="file-name">{{ fileName }}</p>
      </div>
      <ScrollPanel v-if="dataReady" class="scroll-panel">
        <ul class="list experiments-list">
          <li v-for="id in experiments.keys()" :key="id" class="experiments-list-item">
            <Checkbox v-model="selectedExperiments" :inputId="id" name="experiment" :value="id" />
            <label :for="id">{{ id }}</label>
            <Divider />
          </li>
        </ul>
      </ScrollPanel>
    </div>
  </aside>

  <main>
    <ul v-if="selectedExperiments.length > 0" class="list">
      <li v-for="metric in uniqueMetrics" :key="metric" class="chart">
        <Chart type="line" :data="getChartData(metric)" :options="chartOptions" class="h-[30rem]" />
      </li>
    </ul>
  </main>
</template>

<style scoped>
aside .wrapper {
  margin: 0 auto;
  padding: 10px;
  max-width: 375px;
  display: flex;
  flex-direction: column;
  gap: 1rem;
  transition: 0.3s;
}
h1 {
  font-size: 2.5rem;
}
.file-name {
  overflow: hidden;
  text-overflow: ellipsis;
}
.list {
  list-style-type: none;
  display: flex;
  flex-direction: column;
  gap: 1rem;
  padding: 0;
}
.experiments-list {
  gap: 0;
  overflow-y: auto;
}
.experiments-list-item label {
  padding-left: 0.5rem;
}
.scroll-panel {
  height: 50vh;
  padding: 0.8rem 0.8rem 0 0.8rem;
  border-radius: 10px;
  background-color: aliceblue;
}
.scroll-panel .p-scrollpanel-content {
  overflow-y: auto;
}

.input {
  display: flex;
  gap: 1rem;
  align-items: center;
}
.chart {
  width: 100%;
}
@media (min-width: 1024px) {
  aside .wrapper {
    position: fixed;
    top: 0;
    left: 1rem;
    height: 100vh;
  }
  .chart {
    width: 58vw;
  }
  .scroll-panel {
    min-height: 0;
  }
}
</style>
