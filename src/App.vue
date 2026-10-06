<script setup lang="ts">
import { computed, onBeforeUnmount, reactive, ref, type ComputedRef } from 'vue';
import mqtt from 'mqtt';
import DataCard from './components/DataCard.vue';

const kamar = ref("kamar_101");

const topics = reactive([
  `rumah_sakit/${kamar.value}/infus/berat`,
  `rumah_sakit/${kamar.value}/infus/persen`,
  `rumah_sakit/${kamar.value}/infus/status`,
]);
const data = reactive<Record<string, string[]>>({
  berat: [],
  persen: [],
  status: [],
});

const parseNumber = (value: unknown): number => {
  const parsed = Number.parseFloat(String(value ?? '0').replace(',', '.'));
  return Number.isFinite(parsed) ? parsed : 0;
};

const getRecentWeights = (): number[] => {
  return (data.berat ?? [])
    .map(parseNumber)
    .filter((value) => Number.isFinite(value))
    .slice(0, 6);
};

const calculated = reactive<Record<string, ComputedRef<number>>>({
  berat: computed(() => {
    const weights = getRecentWeights();
    if (weights.length < 2) return 0;

    const newest = weights[0] ?? 0;
    const oldest = weights.at(-1) ?? newest;

    return Math.abs(newest - oldest);
  }),
  debit: computed(() => {
    const weights = getRecentWeights();
    if (weights.length < 2) return 0;

    const newest = weights[0] ?? 0;
    const oldest = weights.at(-1) ?? newest;
    const deltaGram = Math.abs(newest - oldest);
    const sampleCount = Math.max(weights.length - 1, 1);
    const avgDeltaPerSample = deltaGram / sampleCount;
    const intervalSeconds = 0.1;

    // 1 g ≈ 1 mL untuk cairan infus, lalu ubah ke L/menit
    return (avgDeltaPerSample / 1000) / (intervalSeconds / 60);
  }),
});

const client = mqtt.connect('wss://broker.hivemq.com:8884/mqtt', {
  clientId: 'vue-client-' + Math.random().toString(16).slice(2),
  clean: true,
  reconnectPeriod: 1000,
  connectTimeout: 15000,
});

client.on('connect', () => {
  topics.forEach((topic) => {
    client.subscribe(topic, (error) => {
      if (error) {
        console.error('Subscribe failed:', error);
        return;
      }
      console.log('Subscribed to:', topic);
    });
  })
});

client.on('message', (receivedTopic, message) => {
  const topic = receivedTopic.split("/").at(-1) as string;

  data[topic]?.unshift(message.toString());
  if((data[topic]?.length ?? 0) > 10) data[topic]?.pop(); 
});

client.on('error', (error) => {console.error('MQTT error:', error); });

onBeforeUnmount(() => {
  client.end();
});
</script>

<template>
  <main class="p-4 flex flex-col max-w-6xl w-full">
    <section class="pb-2 lg:pb-4">
      <h1 class="text-3xl tracking-tighter font-bold text-slate-600">Infuse Dashboard</h1>
      <h2 class="text-xs">Monitoring patient infuse</h2>
    </section>

    <section class="grid grid-cols-3 lg:grid-cols-6 gap-2">
      <DataCard label="Infusion" :data="data.persen?.at(0) ?? 'AMAN'" unit="%" class="bg-zinc-200" :footer="data.status?.at(0)" />
      <DataCard label="Weight" :data="data.berat?.at(0) ?? '0'" unit="g" class="col-span-2 bg-slate-300" :footer="`Changes ${String(((calculated.berat ?? 0) as number).toFixed(2)) ?? '0'}g`" />
      <DataCard label="Flow" :data="(String(((calculated.debit ?? 0) as number).toFixed(2))) ?? '0'" unit="L/min" class="bg-zinc-200" />
      <DataCard label="Room" :data="kamar"  class="col-span-2"/>
    </section>

    <section class="grid grid-cols-1 md:grid-cols-2 mt-2 lg:mt-4">
      <h2 class="sm:col-span-2 md:col-span-3 text-2xl font-semibold text-slate-600">History</h2>

      <ul class="">
        <li v-for="weight, key in data.berat" :key>
          {{ weight }} g
        </li>
      </ul>
      <ul class="">
        <li v-for="persen, key in data.persen" :key>
          {{ persen }} %
        </li>
      </ul>
  </section>
  </main>
</template>

<style scoped></style>
