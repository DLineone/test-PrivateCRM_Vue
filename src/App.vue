<script setup lang="ts">
  import clients from "@/data.json";
  import DietsTable from "@/components/DietsTable.vue"
  import { ref, watch } from "vue";

  let filter = ref<number | "">("");
</script>

<template>
  Начитается через <input v-model="filter" min="0" max="999" type="number"> дней.
  <DietsTable :filter="filter" :clients="clients"/>
</template>

<style scoped>
  input
  {
    margin-block: 20px;
    width: 50px;
  }
</style>
