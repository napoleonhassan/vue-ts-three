<template>
  <main class="container m-auto p-10">
    <Screen 
      :lightIntensity="1"
      :modelUrl="'../src/assets/models/Flywheel.stl'"
      :textureUrl="'../src/assets/materials/basic_1.jpg'"
      :data="deviations"
    />
  </main>
</template>

<script setup lang="ts">
import Screen from "./components/Screen.vue";
import { deviations } from './data/mock';
</script>