<script setup lang="ts">
interface Props {
    label: string
    data: string
    unit?: string
    footer?: string
}

defineProps<Props>()
</script>

<template>
  <div class="rounded-xl border border-slate-300 bg-white p-4 shadow-sm shadow-slate-200">
    <div class="flex items-center justify-between gap-2">
      <span class="text-[10px] font-semibold uppercase tracking-[0.18em] text-slate-500">{{ label }}</span>
      
    </div>

    <div class="mt-4 flex items-end gap-1">
      <span class="text-3xl font-bold tracking-tight text-slate-800">
        {{ data }}
        <span v-if="unit" class="rounded bg-slate-200 px-2 py-1 text-[10px] font-medium uppercase tracking-[0.12em] text-slate-600">
        {{ unit }}
      </span>
      </span>
    </div>

    <div v-if="footer" class="mt-3 border-t border-slate-200 pt-3 text-xs text-slate-500">
      {{ footer }}
    </div>
  </div>
</template>