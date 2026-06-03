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
  <div
    class="demo-iframe-wrap"
    @pointerdown.stop
    @pointermove.stop
    @pointerup.stop
    @touchstart.stop
    @touchmove.stop
    @touchend.stop
  >
    <iframe
      :src="src"
      width="100%"
      :height="height"
      frameborder="0"
      class="rounded-lg shadow-lg"
      allow="fullscreen"
    />
  </div>
</template>

<style scoped>
.demo-iframe-wrap {
  touch-action: none;
}
.demo-iframe-wrap iframe {
  touch-action: none;
}
</style>
