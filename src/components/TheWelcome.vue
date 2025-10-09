<script setup lang="ts">
import { ref } from 'vue'
import PouchDB from 'pouchdb'
const counter = ref(110)
const increment = () => {
  counter.value++
}
</script>
<template>
  <h1>Welcome</h1>
  <p>Counter: {{ counter }}</p>
  <button @click="increment">Increment</button>
</template>
