<script setup lang="ts">
import {Ref, ref, watch, computed, onMounted} from 'vue'
import {PageReferrer, UsersGroupedByStat} from "@backend/api-front/routes/stats";
import TableData, {Column} from "@frontend/src/components/TableData.vue";
import {api, apiWrapper} from "@frontend/src/lib/api";
import {Filter} from "@backend/lib/models/filter";

const loadingSource = ref(false);
const loadingMedium = ref(false);
const loadingCampaign = ref(false);
const loadingTerm = ref(false);
const loadingContent = ref(false);

const columnsSource: Column[] = [
  { name: "Source", type: "string", index: "group", gridColumn: "6fr", canFilter: true },
  { name: "Visitors", type: "number", index: "visitors", gridColumn: "2fr" },
];
const columnsMedium: Column[] = [
  { name: "Medium", type: "string", index: "group", gridColumn: "6fr", canFilter: true },
  { name: "Visitors", type: "number", index: "visitors", gridColumn: "2fr" },
];
const columnsCampaign: Column[] = [
  { name: "Campaign", type: "string", index: "group", gridColumn: "6fr", canFilter: true },
  { name: "Visitors", type: "number", index: "visitors", gridColumn: "2fr" },
];
const columnsTerm: Column[] = [
  { name: "Term", type: "string", index: "group", gridColumn: "6fr", canFilter: true },
  { name: "Visitors", type: "number", index: "visitors", gridColumn: "2fr" },
];
const columnsContent: Column[] = [
  { name: "Content", type: "string", index: "group", gridColumn: "6fr", canFilter: true },
  { name: "Visitors", type: "number", index: "visitors", gridColumn: "2fr" },
];

export interface Props {
  sites: string[],
  fromDate?: Date,
  toDate?: Date,
  filter: Filter
}
const props = withDefaults(defineProps<Props>(), { });

const activeTab = ref('source')

const emit = defineEmits<{
  (e: 'loading', val: boolean): void,
  (e: 'filter-change', val: Filter): void
}>()

let source: Ref<UsersGroupedByStat[] | undefined> = ref();
let medium: Ref<UsersGroupedByStat[] | undefined> = ref();
let campaign: Ref<UsersGroupedByStat[] | undefined> = ref();
let term: Ref<UsersGroupedByStat[] | undefined> = ref();
let content: Ref<UsersGroupedByStat[] | undefined> = ref();

const loading = ref(true);
watch(() => [loading.value], async () => {
  emit('loading', loading.value)
})

async function loadData(groupBy: "utm_source" | "utm_medium" | "utm_campaign" | "utm_term" | "utm_content",
                        loading: Ref<boolean>,
                        data: Ref<UsersGroupedByStat[] | undefined>)
{
  if (props.sites.length === 0 || !props.fromDate || !props.toDate)
    return;

  const resp = await apiWrapper(api.getUsersGroupedByStatForPeriod.query({
    sites: props.sites,
    from: props.fromDate?.toISOString(),
    to: props.toDate?.toISOString(),
    filter: props.filter,
    groupBy
  }), loading);

  if(!resp)
    return;

  data.value = resp;

  // loading.value = true;
  // await new Promise(resolve => setTimeout(resolve, 2000));
  // const mockData =  [
  //   { "group": "desktop", "visitors": 2 },
  //   { "group": "mobile", "visitors": 569 },
  // ]
  // loading.value = false;
  // data.value = mockData;
}
watch(() => [props.sites, props.fromDate, props.toDate, activeTab.value, props.filter], async () => {
  if(activeTab.value === "source")
    await loadData("utm_source", loadingSource, source);
  else if(activeTab.value === "medium")
    await loadData("utm_medium", loadingMedium, medium);
  else if(activeTab.value === "campaign")
    await loadData("utm_campaign", loadingCampaign, campaign);
  else if(activeTab.value === "term")
    await loadData("utm_term", loadingTerm, term);
  else if(activeTab.value === "content")
    await loadData("utm_content", loadingContent, content);
}, {
  deep: true
})

async function refresh() {
  source.value = [];
  medium.value = [];
  campaign.value = [];
  term.value = [];
  content.value = [];

  activeTab.value = "source";
  await loadData("utm_source", loadingSource, source);
}

onMounted(() => {
  /* Fix for Vite HMR that will not fire loadData because the watch will not fire, the data it is watching did not change */
  if(!source.value) {
    activeTab.value = "source";
    loadData("utm_source", loadingSource, source);
  }
})

defineExpose({
  refresh
});

function rowClick(filterKey: string, rowText: any) {
  emit('filter-change', { [filterKey]: rowText })
}
</script>

<template>
  <div class="container">
    <el-tabs v-model="activeTab" tab-position="top" style="margin-top: 10px;">
      <el-tab-pane name="source" label="Source" class="tab-pane">
        <TableData :columns="columnsSource" :rows="source || []" :loading="loadingSource" :page-size="16"
                   @click="(rowText) => rowClick('utm_source', rowText)"></TableData>
      </el-tab-pane>
      <el-tab-pane name="medium" label="Medium" class="tab-pane">
        <TableData :columns="columnsMedium" :rows="medium || []" :loading="loadingMedium" :page-size="16"
                   @click="(rowText) => rowClick('utm_medium', rowText)"></TableData>
      </el-tab-pane>
      <el-tab-pane name="campaign" label="Campaign" class="tab-pane">
        <TableData :columns="columnsCampaign" :rows="campaign || []" :loading="loadingCampaign" :page-size="16"
                   @click="(rowText) => rowClick('utm_campaign', rowText)"></TableData>
      </el-tab-pane>
      <el-tab-pane name="term" label="Term" class="tab-pane">
        <TableData :columns="columnsTerm" :rows="term || []" :loading="loadingTerm" :page-size="16"
                   @click="(rowText) => rowClick('utm_term', rowText)"></TableData>
      </el-tab-pane>
      <el-tab-pane name="content" label="Content" class="tab-pane">
        <TableData :columns="columnsContent" :rows="content || []" :loading="loadingContent" :page-size="16"
                   @click="(rowText) => rowClick('utm_content', rowText)"></TableData>
      </el-tab-pane>
    </el-tabs>
  </div>
</template>

<style scoped>
.tab-pane {
  padding: 0 10px; margin-top: -10px;
}
</style>
