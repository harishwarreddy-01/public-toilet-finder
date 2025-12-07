<template>
  <div class="app-root">
    <main>
      <ToiletMap />
    </main>
  </div>
</template>

<script setup lang="ts">
import ToiletMap from './components/ToiletMap.vue';
</script>

<style>
html, body, #app {
	height: 100%;
	margin: 0;
}
main {
	min-height: 100vh;
}
</style>
