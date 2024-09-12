<template>
  <main class="container m-auto p-10">
    <Screen 
    :lightIntensity="10"
    :modelUrl="'../src/assets/models/gear.gltf.glb'"
    :data="deviations"
    />
  </main>
</template>

<script setup lang="ts">
import Screen from "./components/Screen.vue";
import { deviations } from './data/mock';
</script>