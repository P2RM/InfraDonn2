<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'
import pouchdbFind from 'pouchdb-find'

PouchDB.plugin(pouchdbFind)

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
  created_at: string
  content?: string
}

const localDB = ref<PouchDB.Database | null>(null)
const remoteDB = ref<PouchDB.Database | null>(null)
const storage = ref<PouchDB.Database | null>(null)
const postsData = ref<Post[]>([])

const searchFirstName = ref('')

const newPost = ref<Post>({
  type: 'post',
  name: { first: 'sdf', last: 'dsf' },
  email: 'dsfss@dfs.com',
  tags: [],
  created_at: '',
  content: 'asdsad',
})

const editingPost = ref<Post | null>(null)

const startEdit = (post: Post) => {
  editingPost.value = { ...post }
}
const cancelEdit = () => {
  editingPost.value = null
}

const isPopulating = ref(false)
const populateFactory = async (nb = 100) => {
  if (!storage.value) return
  isPopulating.value = true
  const fakeNames: string[] = [
    'Alice',
    'Bob',
    'Charlie',
    'David',
    'Eva',
    'François',
    'Gus',
    'Henry',
    'Iris',
    'Jack',
    'Karim',
    'Léna',
    'Maria',
    'Nina',
    'Oscar',
    'Paul',
    'Quentin',
    'Rita',
    'Sophie',
    'Tom',
  ]
  const bulkDocs: Post[] = []
  for (let i = 0; i < nb; i++) {
    const firstName = fakeNames[i % fakeNames.length] || 'Anonyme'
    bulkDocs.push({
      type: 'post',
      name: {
        first: firstName + (Math.floor(i / fakeNames.length) || ''),
        last: 'Testeur' + i,
      },
      email: `testeur${i}@exemple.com`,
      tags: ['factory', 'test'],
      created_at: new Date(Date.now() - 1000 * 60 * 60 * 24 * i).toISOString(),
      content: 'Document généré #' + i,
    })
  }
  await storage.value.bulkDocs(bulkDocs)
  fetchData()
  isPopulating.value = false
}

const ensureIndex = async () => {
  if (!storage.value) return
  try {
    await (storage.value as any).createIndex({
      index: { fields: ['name.first'] },
      name: 'firstName-index',
    })
  } catch (e) {
    console.error('Erreur index', e)
  }
}

const searchByFirstName = async () => {
  if (!storage.value) return
  await ensureIndex()
  const filter = searchFirstName.value.trim()
  if (!filter) {
    fetchData()
    return
  }
  try {
    const result = await (storage.value as any).find({
      selector: {
        'name.first': { $regex: `(?i)^${filter}` },
      },
      use_index: 'firstName-index',
    })
    postsData.value = result.docs
  } catch (e) {
    console.error('Recherche impossible', e)
  }
}

const initDatabase = () => {
  localDB.value = new PouchDB('local_db')
  remoteDB.value = new PouchDB('http://admin:admin@127.0.0.1:5984/test_database')
  storage.value = localDB.value
  ensureIndex()
  console.log('Bases locale et distante initialisées')
}

const fetchData = () => {
  if (!storage.value) {
    console.warn('Base de données non initialisée')
    return
  }
  storage.value
    .allDocs({ include_docs: true })
    .then((result) => {
      postsData.value = result.rows.map((row) => row.doc as Post).filter((doc) => !!doc)
      console.log(postsData.value)
      //postsData.value.sort((a, b) => (a.name.first > b.name.first ? 1 : -1))
      console.log('✅ Données récupérées :', postsData.value)
    })
    .catch((error) => {
      console.error('❌ Erreur lors de la récupération :', error)
    })
}

const deleteDocument = (docId: string, docRev: string) => {
  if (!storage.value) {
    console.warn('Base de données non initialisée')
    return
  }
  storage.value
    .remove(docId, docRev)
    .then((response) => {
      console.log('✅ Document supprimé :', response)
      fetchData()
    })
    .catch((error) => {
      console.error('❌ Erreur lors de la suppression du document :', error)
    })
}

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
      fetchData()
      editingPost.value = null
    })
    .catch((error) => {
      console.error('❌ Erreur lors de la mise à jour du document :', error)
    })
}

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
    name: {
      first: newPost.value.name.first.trim(),
      last: newPost.value.name.last ? newPost.value.name.last.trim() : '',
    },
    email: newPost.value.email.trim(),
    tags: newPost.value.tags.length ? newPost.value.tags : ['vue', 'pouchdb'],
    created_at: new Date().toISOString(),
    content: newPost.value.content,
  }
  storage.value
    .post(postToAdd)
    .then((response) => {
      console.log('✅ Document ajouté :', response)
      fetchData()
      newPost.value = {
        type: 'post',
        name: { first: '', last: '' },
        email: '',
        tags: [],
        created_at: '',
        content: '',
      }
    })
    .catch((error) => {
      console.error("❌ Erreur lors de l'ajout du document :", error)
    })
}

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
  initDatabase()
  fetchData()
})
</script>

<template>
  <h1>Base de données PouchDB</h1>

  <div style="margin-bottom: 1rem; padding: 0.5rem; background: #f7f7ff; border-radius: 8px">
    <button @click="populateFactory()" :disabled="isPopulating">Générer 100 documents</button>
    <span v-if="isPopulating" style="margin-left: 1rem">Chargement...</span>
    <button @click="ensureIndex" style="margin-left: 1rem">Créer l'index prénom (si besoin)</button>
    <input
      v-model="searchFirstName"
      @input="searchByFirstName"
      style="margin-left: 1rem; padding: 0.25rem; border: 1px solid #ccc"
      placeholder="🔍 Rechercher par prénom"
    />
    <button
      v-if="searchFirstName"
      @click="
        () => {
          searchFirstName = ''
          fetchData()
        }
      "
      style="margin-left: 6px"
    >
      ❌
    </button>
  </div>

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
    <div style="margin-bottom: 0.5rem">
      <label>Contenu :</label>
      <textarea v-model="newPost.content" rows="2" placeholder="Entrez un contenu"></textarea>
    </div>
    <button type="submit">Ajouter</button>
  </form>

  <div class="flex gap-2 my-3">
    <button role="button" @click="fetchData">🔄 Rafraîchir</button>
    <button role="button" @click="syncDatabases">🔁 Synchroniser</button>
  </div>

  <article v-for="post in postsData" :key="post._id">
    <div v-if="!editingPost || editingPost._id !== post._id">
      <h2>{{ post.name.first }} {{ post.name.last }}</h2>
      <p>{{ post.email }}</p>
      <p>Tags : {{ post.tags.join(', ') }}</p>
      <p>Créé le : {{ post.created_at }}</p>
      <p v-if="post.content">Contenu : {{ post.content }}</p>
      <button role="button" @click="startEdit(post)">✏️ Modifier</button>
      <button role="button" @click="deleteDocument(post._id!, post._rev!)">🗑️ Supprimer</button>
    </div>
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
      <div style="margin-bottom: 0.5rem">
        <label>Contenu :</label>
        <textarea
          v-model="editingPost.content"
          rows="2"
          placeholder="Modifier le contenu"
        ></textarea>
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
input,
textarea {
  margin-left: 0.5rem;
  padding: 0.25rem;
}
button {
  padding: 0.5rem 1rem;
  cursor: pointer;
  margin-right: 0.5rem;
}
</style>
