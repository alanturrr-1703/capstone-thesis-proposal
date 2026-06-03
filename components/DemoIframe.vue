<script setup lang="ts">
const props = withDefaults(defineProps<{
  demo: string
  height?: string
}>(), {
  height: '340px',
})

const base = import.meta.env.BASE_URL || '/'
const src = `${base}${props.demo}`
</script>

<template>
  <iframe
    :src="src"
    width="100%"
    :height="height"
    frameborder="0"
    class="rounded-lg shadow-lg"
  />
</template>
