<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'

declare interface Post {
  _id?: string
  _rev?: string
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
const storage = ref<PouchDB.Database | null>(null)
// Données stockées
const postsData = ref<Post[]>([])

// Initialisation de la base de données
const initDatabase = () => {
  console.log('=> Connexion à la base de données')
  const db = new PouchDB('http://admin:admin@127.0.0.1:5984/test_database')
  if (db) {
    console.log('Connecté à la collection : ' + db.name)
    storage.value = db
  } else {
    console.warn('Échec lors de la connexion à la base de données')
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
      // On met à jour les données réactives avec les docs
      postsData.value = result.rows.map((row) => row.doc).filter((doc): doc is Post => !!doc)
      console.log('✅ Données récupérées :', postsData.value)
    })
    .catch((error) => {
      console.error('❌ Erreur lors de la récupération :', error)
    })
}

// ➕ Ajout d’un document à la base de données
const addDocument = (newPost: Post) => {
  if (!storage.value) {
    console.warn('Base de données non initialisée')
    return
  }

  // On enlève les champs _id et _rev s’ils existent
  const { _id, _rev, ...docData } = newPost

  storage.value
    .post(docData)
    .then((response) => {
      console.log('✅ Document ajouté :', response)
      fetchData() // Rafraîchit la datatable après ajout
    })
    .catch((error) => {
      console.error('❌ Erreur lors de l’ajout du document :', error)
    })
}

onMounted(() => {
  console.log('=> Composant initialisé')
  initDatabase()
  fetchData()
})
</script>

<template>
  <h1>Base de données PouchDB</h1>

  <!-- Boutons d’action -->
  <div class="flex gap-2 my-3">
    <button role="button" @click="fetchData">🔄 Rafraîchir</button>
    <button
      role="button"
      @click="
        addDocument({
          type: 'post',
          name: { first: 'Jean', last: 'Dupont' },
          email: 'jean.dupont@example.com',
          tags: ['vue', 'pouchdb'],
          created_at: new Date().toISOString(),
        })
      "
    >
      ➕ Ajouter un document
    </button>
  </div>

  <!-- Liste des documents -->
  <article v-for="post in postsData" :key="post._id">
    <h2>{{ post.name.first }} {{ post.name.last }}</h2>
    <p>{{ post.email }}</p>
    <p>Tags : {{ post.tags.join(', ') }}</p>
    <p>Créé le : {{ post.created_at }}</p>
    <hr />
  </article>
</template>

<!-- <script setup lang="ts">
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
  const db = new PouchDB('http://admin:admin@127.0.0.1:5984/test_database')
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
</template> -->
