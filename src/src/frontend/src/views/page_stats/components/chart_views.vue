<script setup lang="ts">
import {Ref, ref, watch, onMounted} from 'vue'
import {api, apiWrapper} from "@frontend/src/lib/api";
import {ChartView} from "@backend/api-front/routes/stats";
// import Plotly from 'Plotly'
// import * as Plotly, {PlotData} from "plotly.js";
//@ts-ignore
import * as Plotly from "plotly.js-dist-min";
import { ArrowDown } from '@element-plus/icons-vue'
import {useDark} from "@vueuse/core";
import {DateUtils} from "@frontend/src/lib/date_utils";
import {groupBy} from "lodash";
import {Filter} from "@backend/lib/models/filter";

export interface Props {
  sites: string[],
  fromDate?: Date,
  toDate?: Date,
  filter: Filter
}
const props = withDefaults(defineProps<Props>(), { });

const emit = defineEmits<{
  (e: 'loading', val: boolean): void,
}>()

let chartViews: Ref<ChartView[] | undefined> = ref();

const loading = ref(true);
watch(() => [loading.value], async () => {
  emit('loading', loading.value)
})

function* getDatesInRange(startDate: string, endDate: string, hourly: boolean): Generator<string> {
  const currentDate = new Date(startDate);
  while (currentDate <= new Date(endDate)) {
    yield currentDate.toISOString();
    if (hourly) {
      currentDate.setHours(currentDate.getHours() + 1);
    } else {
      currentDate.setDate(currentDate.getDate() + 1);
    }
  }
}

function fillMissingDates(chartData: ChartView[], minDate: string, maxDate: string, sites: string[], hourly: boolean): ChartView[] {
  const filledData: ChartView[] = [];

  for (const site of sites) {
    const siteData = chartData.filter((entry) => entry.site === site);
    for (const date of getDatesInRange(minDate, maxDate, hourly)) {
      const entry = siteData.find((x) => x.date_key === date);
      if (!entry) {
        filledData.push({
          site,
          date_key: date,
          visitors: 0,
          views: 0,
        });
      } else {
        filledData.push(entry);
      }
    }
  }

  return filledData;
}



async function loadData()
{
  if (props.sites.length === 0 || !props.fromDate || !props.toDate)
    return;

  // //
  // // TODO: now auto, later make this configurable in the settings pane + the chart type
  const daysBetween = DateUtils.daysBetween(props.fromDate, props.toDate);
  let period: "hour" | "day" = "day";
  if(daysBetween < 3)
    period = "hour";

  const timeZone = DateUtils.currentTimeZone();
  const resp = await apiWrapper(api.getChartViews.query({
    sites: props.sites,
    from: props.fromDate?.toISOString(),
    to: props.toDate?.toISOString(),
    filter: props.filter,
    period,
    timeZone
  }), loading);

  if (!resp)
    return;

  const filledInData = fillMissingDates(resp, props.fromDate.toISOString(), props.toDate.toISOString(),
    props.sites, period === "hour");
  chartViews.value = filledInData;

  // // loading.value = true;
  // // await new Promise(resolve => setTimeout(resolve, 1000));
  //daily
  // const data = [
  //   // {
  //   //   "site": "tests1",
  //   //   "date_key": "2023-04-23T22:00:00.000Z",
  //   //   "visitors": 1,
  //   //   "views": 144100
  //   // },
  //   {
  //     "site": "tests1",
  //     "date_key": "2023-04-24T22:00:00.000Z",
  //     "visitors": 1,
  //     "views": 144000
  //   },
  //
  //   // {
  //   //   "site": "tests1",
  //   //   "date_key": "2023-04-23T22:00:00.000Z",
  //   //   "visitors": 1,
  //   //   "views": 144000
  //   // },
  //
  //   {
  //     "site": "tests1",
  //     "date_key": "2023-04-26T22:00:00.000Z",
  //     "visitors": 1,
  //     "views": 144000
  //   },
  //   {
  //     "site": "tests1",
  //     "date_key": "2023-04-27T22:00:00.000Z",
  //     "visitors": 1,
  //     "views": 144000
  //   },
  //   {
  //     "site": "tests1",
  //     "date_key": "2023-04-28T22:00:00.000Z",
  //     "visitors": 2,
  //     "views": 144001
  //   },
  //   {
  //     "site": "tests1",
  //     "date_key": "2023-04-29T22:00:00.000Z",
  //     "visitors": 1,
  //     "views": 144000
  //   },
  //   {
  //     "site": "tests1",
  //     "date_key": "2023-04-30T22:00:00.000Z",
  //     "visitors": 1,
  //     "views": 12000
  //   },
  //
  //
  //   {
  //     "site": "tests2",
  //     "date_key": "2023-04-23T22:00:00.000Z",
  //     "visitors": 1,
  //     "views": 2356
  //   },
  //   {
  //     "site": "tests2",
  //     "date_key": "2023-04-24T22:00:00.000Z",
  //     "visitors": 1,
  //     "views": 643
  //   },
  //
  //
  //   {
  //     "site": "example.com",
  //     "date_key": "2023-04-24T22:00:00.000Z",
  //     "visitors": 1,
  //     "views": 2356
  //   },
  //   {
  //     "site": "example.com",
  //     "date_key": "2023-04-25T22:00:00.000Z",
  //     "visitors": 1,
  //     "views": 777
  //   },
  //   {
  //     "site": "example.com",
  //     "date_key": "2023-04-26T22:00:00.000Z",
  //     "visitors": 1,
  //     "views": 12425
  //   },
  //
  // ];
  // // loading.value = false;

  // //hourly
  // const data = [
  //   {
  //     "site": "tests1",
  //     "date_key": "2023-04-27T22:00:00.000Z",
  //     "visitors": 1,
  //     "views": 144100
  //   },
  //   {
  //     "site": "tests1",
  //     "date_key": "2023-04-27T23:00:00.000Z",
  //     "visitors": 1,
  //     "views": 144000
  //   },
  //
  //   {
  //     "site": "tests1",
  //     "date_key": "2023-04-28T00:00:00.000Z",
  //     "visitors": 1,
  //     "views": 144000
  //   },
  //
  //   {
  //     "site": "tests1",
  //     "date_key": "2023-04-28T01:00:00.000Z",
  //     "visitors": 1,
  //     "views": 144000
  //   },
  //   {
  //     "site": "tests1",
  //     "date_key": "2023-04-28T02:00:00.000Z",
  //     "visitors": 1,
  //     "views": 144000
  //   },
  //   {
  //     "site": "tests1",
  //     "date_key": "2023-04-28T03:00:00.000Z",
  //     "visitors": 2,
  //     "views": 144001
  //   },
  //   {
  //     "site": "tests1",
  //     "date_key": "2023-04-28T04:00:00.000Z",
  //     "visitors": 1,
  //     "views": 144000
  //   },
  //   {
  //     "site": "tests1",
  //     "date_key": "2023-04-28T05:00:00.000Z",
  //     "visitors": 1,
  //     "views": 12000
  //   },
  //
  //
  //   {
  //     "site": "tests2",
  //     "date_key": "2023-04-28T14:00:00.000Z",
  //     "visitors": 1,
  //     "views": 2356
  //   },
  //   {
  //     "site": "tests2",
  //     "date_key": "2023-04-28T15:00:00.000Z",
  //     "visitors": 1,
  //     "views": 643
  //   },
  //
  // ];

  // console.log("FROM:", props.fromDate.toISOString(), "TO", props.toDate.toISOString());
  // const filledIn = fillMissingDates(data, props.fromDate.toISOString(), props.toDate.toISOString(),
  //   props.sites, period === "hour");
  // console.log(filledIn);
}
watch(() => [props.sites, props.fromDate, props.toDate, props.filter], async () => {
  await loadData();
}, {
  deep: true
})
async function refresh() {
  await loadData();
}
defineExpose({
  refresh
});


const isDark = useDark();
const colors = {
  dark: {
    chartBackground: "#1c1b22",
    chartForeground: "#fff"
  },
  light: {
    chartBackground: "#fff",
    chartForeground: "#1c1b22"
  }
};

const chartRef = ref();
function getGraph(valueKey: "views" | "visitors", chartViews: ChartView[], graphName: string)
{
  const dataX: string[] = [];
  const dataY: number[] = [];
  for(let row of chartViews)
  {
    const date = DateUtils.parseIso(row.date_key);
    const dateInTz = DateUtils.stringifyFormat(date, "yyyy-MM-dd HH");
    dataX.push(dateInTz);
    dataY.push(row[valueKey]);
  }
  const graph: Partial<Plotly.PlotData> = {
    name: graphName,
    x: dataX,
    y: dataY,
    // type: "bar",
    type: "bar",
    mode: 'lines+markers',
    hoverinfo: "x+y+name",
  };
  return graph;
}

function paintChart()
{
  const chartBackground = isDark.value ? colors.dark.chartBackground : colors.light.chartBackground;
  const chartForeground =  isDark.value ? colors.dark.chartForeground : colors.light.chartForeground;

  const groupedViews = groupBy(chartViews.value, "site");

  const graphs: Partial<Plotly.PlotData>[] = [];
  for(let [site, groupedChartViews] of Object.entries(groupedViews))
  {
    let graph:  Partial<Plotly.PlotData>;
    if(chartType.value === "Views")
      graph = getGraph("views", groupedChartViews, site);
    else
      graph = getGraph("visitors", groupedChartViews, site);

    graphs.push(graph);
  }

  // let graph:  Partial<Plotly.PlotData>;
  // if(chartType.value === "Views")
  //   graph = getGraph("views", "Views");
  // else
  //   graph = getGraph("visitors", "Visitors");

  Plotly.newPlot(chartRef.value, graphs,
    {
      barmode: 'stack',
      plot_bgcolor: chartBackground,
      paper_bgcolor: chartBackground,
      legend: {
        font: {
          color: chartForeground
        },
      },
      title: {
        font: {
          color: chartForeground
        }
      },
      scene: {
        xaxis: {
          color: chartForeground
        },
        yaxis: {
          color: chartForeground
        },
        zaxis: {
          color: chartForeground
        },
      },
      hoverlabel: {
        font: {
          color: chartForeground
        }
      },
      showlegend: false,
      autosize: true,
      margin: {
        l: 20,
        r: 0,
        b: 40,
        t: 10,
        pad: 5
      },
      hovermode: "x unified",
      xaxis : {
        color: chartForeground,
        range: [props.fromDate!.getTime(),props.toDate!.getTime()],
        showgrid: false,
        zeroline: false,
        showline: false,
      },
      yaxis : {
        color: chartForeground,
        automargin: true,
        fixedrange: true,
        rangemode: "tozero", //nonnegative
        // zeroline: true,
      },
    }, {
      responsive: true,
      displayModeBar: false,
    });
  Plotly.Plots.resize(chartRef.value); /* Because div is hidden, plotly can not calculate width */
}

type ChartType = "Views" | "Visitors";
const chartType: Ref<ChartType> = ref("Views");
function changeChartType(command: ChartType)
{
  chartType.value = command;
  console.log("changeChartType", command)
  paintChart();
}

onMounted(() => {
  watch([isDark, chartViews, chartType], () => {
    paintChart();
  });

  /* Fix for Vite HMR that will not fire loadData because the watch will not fire, the data it is watching did not change */
  if(!chartViews.value)
    loadData();
})
</script>

<template>
  <div class="container" style="height: 470px;">
    <div v-show="!loading" style="position: relative; padding: 10px;">
<!--    <div v-if="false" style="position: relative; padding: 10px;">-->
      <el-dropdown class="chart_selector" trigger="click" @command="changeChartType">
      <span>
        {{chartType}}
        <el-icon style="top: 2px;"> <arrow-down /> </el-icon>
      </span>
        <template #dropdown>
          <el-dropdown-menu>
            <el-dropdown-item command="Views">Views</el-dropdown-item>
            <el-dropdown-item command="Visitors">Visitors</el-dropdown-item>
          </el-dropdown-menu>
        </template>
      </el-dropdown>

      <div class="chart" ref="chartRef"></div>
    </div>
    <el-skeleton v-show="loading" :loading="true" animated style="height: 100%">
      <template #template>
        <el-skeleton-item variant="image" style="height: 91%; margin: 20px;" />
      </template>
    </el-skeleton>
  </div>
</template>

<style scoped>

.chart_selector {
  /*margin-top: 5px;*/
  font-weight: bold;
  cursor: pointer;
  outline: none;

  position: absolute;
  top: 0;
  right: 0;
  z-index: 1000;
}
.chart_selector span {
  outline: none;
  padding: 10px;
  backdrop-filter: blur(1rem);
}



</style>
