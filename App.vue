<script setup>
import { computed, onMounted, reactive, ref } from "vue";
import axios from 'axios';

const LIVE_STREAMS_URL = "https://raw.githubusercontent.com/drmlive/fancode-live-events/refs/heads/main/fancode.json";
const data = reactive({
  isLoading: false,
  error: null,
  matches: [],
})
const ALL = "ALL";
const selectedCategory = ref(ALL);

const fetchLiveStreams = async () => {
  try {
    data.isLoading = true;
    const res = await axios.get(LIVE_STREAMS_URL);
    if (res.status == 200 && res.data) {
      data.matches = res.data.matches;
    }
  } catch (err) {
    console.error(err)
    data.error = err;
  } finally {
    data.isLoading = false;
  }
}

const categorizedData = computed(() => {
  const d = new Map();

  if (data.isLoading || data.error) {
    return d;
  }

  for (let i = 0; i < data.matches.length; i++) {
    const key = data.matches[i].event_category;
    if (d.has(key)) {
      d.set(key, [
        ...d.get(key),
        data.matches[i],
      ])
    } else {
      d.set(key, [data.matches[i]])
    }
  }

  d.forEach((v, k, m) => {
    v.sort((a, b) => {
      if (a.status == "LIVE" && b.status == "UPCOMING") return -1;
      if (a.status == "UPCOMING" && b.status == "LIVE") return 1;

      const ad = new Date(a.startTime);
      const bd = new Date(b.startTime);

      return ad.getTime() - bd.getTime();
    })
  })

  return d
})

const filteredCategorizedData = computed(() => {
  if (selectedCategory.value == ALL) {
    return categorizedData.value;
  }

  return new Map(
    [...categorizedData.value.entries()].filter(([k]) => k === selectedCategory.value)
  )
})

onMounted(fetchLiveStreams);
</script>

<template>
  <div class="p-2">
    <div class="flex justify-between items-center">
      <button class="cursor-pointer" @click="fetchLiveStreams" :disabled="data.isLoading">Refresh</button>

      <select v-model="selectedCategory">
        <option :value="ALL">{{ALL}}</option>
        <option v-for="[cat] in categorizedData" :value="cat">{{ cat }}</option>
      </select>
    </div>

    <div v-if="data.isLoading">Loading ...</div>
    <div v-else class="w-full">
      <div v-for="[cat, matches] in filteredCategorizedData">
        <div class="py-2 border-b mb-2 font-bold">{{ cat }}</div>
        <div v-for="match in matches">
          <a :href="match.adfree_url" target="_blank">
            [{{ match.status }}] - {{ match.title }} - {{ match.startTime }}
          </a>
        </div>
      </div>
    </div>
  </div>
</template>
