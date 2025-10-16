<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'

declare interface Post {
  _id: string
  _rev: string
  type: string
  name: {
    first: string
    last: string
  }
  email: string
  tags: string[]
  created_at: string // ou Date
}

// Référence à la base de données
const storage = ref()
// Données stockées
const postsData = ref<Post[]>([])

// Initialisation de la base de données
const initDatabase = () => {
  console.log('=> Connexion à la base de données')
  const db = new PouchDB('http://admin:admin@127.0.0.1:5986/test_database')
  if (db) {
    console.log('Connecté à la collection : ' + db?.name)
    storage.value = db
  } else {
    console.warn('Echec lors de la connexion à la base de données')
  }
}

// Récupération des données
const fetchData = () => {
  if (!storage.value) {
    console.warn('Base de données non initialisée')
    return
  }
  storage.value
    .allDocs({ include_docs: true })
    .then((result) => {
      console.log(result)
    })
    .catch((error) => {
      console.log(error)
    })
}

onMounted(() => {
  console.log('=> Composant initialisé')
  initDatabase()
  //fetchData()
})
</script>

<template>
  <h1>Fetch Data</h1>
  <article v-for="post in postsData" v-bind:key="(post as any).id">
    <h2>{{ post.name.first }}</h2>
    <p>{{ post.name.last }}</p>
    <p>{{ post.email }}</p>
    <p>{{ post.tags.join(', ') }}</p>
    <p>{{ post.created_at }}</p>
  </article>
  <button role="button" @click="fetchData">Fetch</button>
</template>
