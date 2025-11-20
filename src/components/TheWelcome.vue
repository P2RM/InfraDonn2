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

// AJOUT : bases locale/distante
const localDB = ref<PouchDB.Database | null>(null)
const remoteDB = ref<PouchDB.Database | null>(null)

// Référence à la base de données
const storage = ref<PouchDB.Database | null>(null)
// Données stockées
const postsData = ref<Post[]>([])

// Modèle pour le formulaire d'ajout
const newPost = ref<Post>({
  type: 'post',
  name: { first: '', last: '' },
  email: '',
  tags: [],
  created_at: '',
})

// Modèle pour le formulaire de modification
const editingPost = ref<Post | null>(null)

// Activer le mode édition
const startEdit = (post: Post) => {
  editingPost.value = { ...post }
}

// Annuler l'édition
const cancelEdit = () => {
  editingPost.value = null
}

// AJOUT : Initialisation des bases
const initDatabase = () => {
  localDB.value = new PouchDB('local_db')
  remoteDB.value = new PouchDB('http://admin:admin@127.0.0.1:5984/test_database')
  storage.value = localDB.value // on pointe storage vers la locale
  console.log('Bases locale et distante initialisées')
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
      postsData.value = result.rows.map((row) => row.doc as Post).filter((doc) => !!doc)
      console.log('✅ Données récupérées :', postsData.value)
    })
    .catch((error) => {
      console.error('❌ Erreur lors de la récupération :', error)
    })
}

// 🗑️ Suppression d'un document de la base de données
const deleteDocument = (docId: string, docRev: string) => {
  if (!storage.value) {
    console.warn('Base de données non initialisée')
    return
  }

  storage.value
    .remove(docId, docRev)
    .then((response) => {
      console.log('✅ Document supprimé :', response)
      fetchData() // Rafraîchit la liste après suppression
    })
    .catch((error) => {
      console.error('❌ Erreur lors de la suppression du document :', error)
    })
}

// ✏️ Mise à jour d'un document dans la base de données
const updateDocument = () => {
  if (!storage.value) {
    console.warn('Base de données non initialisée')
    return
  }

  if (!editingPost.value || !editingPost.value._id || !editingPost.value._rev) {
    console.warn('⚠️ Le document doit avoir un _id et un _rev pour être mis à jour')
    return
  }

  storage.value
    .put(editingPost.value)
    .then((response) => {
      console.log('✅ Document mis à jour :', response)
      fetchData() // Rafraîchit la liste après mise à jour
      editingPost.value = null // Ferme le formulaire d'édition
    })
    .catch((error) => {
      console.error('❌ Erreur lors de la mise à jour du document :', error)
    })
}

// ➕ Ajout d'un document à la base de données via formulaire
const addDocument = () => {
  if (!storage.value) {
    console.warn('Base de données non initialisée')
    return
  }

  if (!newPost.value.name.first || !newPost.value.email) {
    alert('Prénom et email obligatoires')
    return
  }

  const postToAdd: Post = {
    type: 'post',
    name: { ...newPost.value.name },
    email: newPost.value.email,
    tags: newPost.value.tags.length ? newPost.value.tags : ['vue', 'pouchdb'],
    created_at: new Date().toISOString(),
  }

  // On enlève les champs _id et _rev s'ils existent
  storage.value
    .post(postToAdd)
    .then((response) => {
      console.log('✅ Document ajouté :', response)
      fetchData() // Rafraîchit la datatable après ajout
      // Réinitialiser le formulaire
      newPost.value = {
        type: 'post',
        name: { first: '', last: '' },
        email: '',
        tags: [],
        created_at: '',
      }
    })
    .catch((error) => {
      console.error("❌ Erreur lors de l'ajout du document :", error)
    })
}

// AJOUT : Synchronisation locale <=> distante
const syncDatabases = () => {
  if (!localDB.value || !remoteDB.value) {
    alert('Bases non initialisées')
    return
  }
  localDB.value
    .sync(remoteDB.value, {
      live: false,
      retry: false,
    })
    .on('complete', (info) => {
      console.log('✅ Synchronisation complète:', info)
      fetchData()
      alert('Synchronisation terminée')
    })
    .on('error', (err) => {
      console.error('❌ Erreur lors de la synchronisation:', err)
      alert('Erreur lors de la synchronisation')
    })
}

onMounted(() => {
  console.log('=> Composant initialisé')
  initDatabase() // AJOUT : initialise local & distant
  fetchData()
})
</script>

<template>
  <h1>Base de données PouchDB</h1>

  <!-- Formulaire d'ajout -->
  <form
    @submit.prevent="addDocument"
    style="margin-bottom: 2rem; padding: 1rem; border: 1px solid #ccc; border-radius: 8px"
  >
    <h3>➕ Ajouter un document</h3>
    <div style="margin-bottom: 0.5rem">
      <label>Prénom :</label>
      <input v-model="newPost.name.first" required />
    </div>
    <div style="margin-bottom: 0.5rem">
      <label>Nom :</label>
      <input v-model="newPost.name.last" />
    </div>
    <div style="margin-bottom: 0.5rem">
      <label>Email :</label>
      <input v-model="newPost.email" type="email" required />
    </div>
    <button type="submit">Ajouter</button>
  </form>

  <!-- Boutons d'action -->
  <div class="flex gap-2 my-3">
    <button role="button" @click="fetchData">🔄 Rafraîchir</button>
    <!-- AJOUT : bouton Synchroniser -->
    <button role="button" @click="syncDatabases">🔁 Synchroniser</button>
  </div>

  <!-- Liste des documents -->
  <article v-for="post in postsData" :key="post._id">
    <!-- Mode affichage normal -->
    <div v-if="!editingPost || editingPost._id !== post._id">
      <h2>{{ post.name.first }} {{ post.name.last }}</h2>
      <p>{{ post.email }}</p>
      <p>Tags : {{ post.tags.join(', ') }}</p>
      <p>Créé le : {{ post.created_at }}</p>
      <button role="button" @click="startEdit(post)">✏️ Modifier</button>
      <button role="button" @click="deleteDocument(post._id!, post._rev!)">🗑️ Supprimer</button>
    </div>

    <!-- Mode édition -->
    <div
      v-else
      style="padding: 1rem; border: 2px solid #007bff; border-radius: 8px; background: #f0f8ff"
    >
      <h3>✏️ Modifier le document</h3>
      <div style="margin-bottom: 0.5rem">
        <label>Prénom :</label>
        <input v-model="editingPost.name.first" required />
      </div>
      <div style="margin-bottom: 0.5rem">
        <label>Nom :</label>
        <input v-model="editingPost.name.last" />
      </div>
      <div style="margin-bottom: 0.5rem">
        <label>Email :</label>
        <input v-model="editingPost.email" type="email" required />
      </div>
      <button role="button" @click="updateDocument()">✅ Enregistrer</button>
      <button role="button" @click="cancelEdit()">❌ Annuler</button>
    </div>
    <hr />
  </article>
</template>

<style scoped>
h1 {
  color: #333;
  margin-bottom: 1rem;
}
input {
  margin-left: 0.5rem;
  padding: 0.25rem;
}
button {
  padding: 0.5rem 1rem;
  cursor: pointer;
  margin-right: 0.5rem;
}
</style>
