<script setup lang="ts">
import {Ref, ref, watch, computed, onMounted} from 'vue'
import {PageReferrer} from "@backend/api-front/routes/stats";
import {Filter} from "@backend/lib/models/filter";
import TableData, {Column} from "@frontend/src/components/TableData.vue";
import {api, apiWrapper} from "@frontend/src/lib/api";

export interface Props {
  sites: string[],
  fromDate?: Date,
  toDate?: Date,
  filter: Filter
}
const props = withDefaults(defineProps<Props>(), { });

const emit = defineEmits<{
  (e: 'loading', val: boolean): void,
  (e: 'filter-change', val: Filter): void
}>()

let pageReferrers: Ref<PageReferrer[] | undefined> = ref();

const loading = ref(true);
watch(() => [loading.value], async () => {
  emit('loading', loading.value)
})

async function loadData() {
  if (props.sites.length === 0 || !props.fromDate || !props.toDate)
    return;

  const resp = await apiWrapper(api.getPageReferrers.query({
    sites: props.sites,
    from: props.fromDate?.toISOString(),
    to: props.toDate?.toISOString(),
    filter: props.filter,
  }), loading);
  if (!resp)
    return;

  pageReferrers.value = resp;

  // loading.value = true;
  // await new Promise(resolve => setTimeout(resolve, 2000));
  // const data =  [
  //   { referrer: "No Referrer", views: 1231243 },
  //   { referrer: "something.com", views: 2 },
  // ]
  // loading.value = false;
  // pageReferrers.value = data;
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

onMounted(() => {
  /* Fix for Vite HMR that will not fire loadData because the watch will not fire, the data it is watching did not change */
  if(!pageReferrers.value)
    loadData();
})

const columns: Column[] = [
  { name: "Referrer", type: "string", index: "referrer", gridColumn: "6fr", canFilter: true, openExternalColumn: "referrer" },
  { name: "Views", type: "number", index: "views", gridColumn: "2fr" },
];

function rowClick(rowText: any) {
  if(rowText === "No Referrer")
    rowText = null;
  emit('filter-change', { referrer: rowText })
}
</script>

<template>
  <div class="container">
    <TableData :columns="columns" :rows="pageReferrers || []" :loading="loading" :page-size="16"
               @click="rowClick"></TableData>
  </div>
</template>

<style scoped>

</style>
