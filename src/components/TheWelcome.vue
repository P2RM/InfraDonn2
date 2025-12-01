<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'
import pouchdbFind from 'pouchdb-find'
import findPlugin from 'pouchdb-find'

PouchDB.plugin(findPlugin)
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
  likes?: number
}

declare interface Comment {
  _id?: string
  _rev?: string
  type: 'comment'
  postId: string
  author: string
  content: string
  created_at: string
}

const localDB = ref<PouchDB.Database | null>(null)
const remoteDB = ref<PouchDB.Database | null>(null)
const storage = ref<PouchDB.Database | null>(null)
const postsData = ref<Post[]>([])
const factoryCount = ref(100)
const searchFirstName = ref('')
const searchQuery = ref('')
const isOnline = ref(true)

const newPost = ref<Post>({
  type: 'post',
  name: { first: '', last: '' },
  email: '',
  tags: [],
  created_at: '',
  content: '',
})

const editingPost = ref<Post | null>(null)

const newCommentContent = ref('')
const commentAuthor = ref('')
const commentEditing = ref<{ _id: string; content: string } | null>(null)
const commentList = ref<Comment[]>([])
const currentPostId = ref<string | null>(null)
const isPopulating = ref(false)

// Premier commentaire par post (clé = postId)
const firstCommentsByPost = ref<Record<string, Comment | null>>({})

const toggleMode = () => {
  isOnline.value = !isOnline.value
  storage.value = isOnline.value ? remoteDB.value : localDB.value
  fetchData()
}

const populateFactory = async (nb = 100) => {
  if (!storage.value) return
  isPopulating.value = true
  try {
    const fakeNames: string[] = [
      /* ... tes prénoms ... */
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
        likes: Math.floor(Math.random() * 20),
      })
    }
    console.log('👉 bulkDocs start', nb)
    const res = await storage.value.bulkDocs(bulkDocs)
    console.log('✅ bulkDocs ok', res)
    await fetchData()
  } catch (err) {
    console.error('❌ bulkDocs error', err)
  } finally {
    isPopulating.value = false
  }
}

const ensureIndex = async () => {
  if (!storage.value) return
  try {
    await (storage.value as any).createIndex({
      index: { fields: ['name.first'] },
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
    postsData.value = result.docs as Post[]
  } catch (e) {
    console.error('Recherche impossible', e)
  }
}

const initDatabase = () => {
  localDB.value = new PouchDB('local_db')
  remoteDB.value = new PouchDB('http://admin:admin@127.0.0.1:5984/test_database')
  storage.value = isOnline.value ? remoteDB.value : localDB.value
  ensureIndex()
  console.log('Bases locale et distante initialisées')
}

const deleteAllDocuments = async () => {
  if (!storage.value) {
    console.warn('Base de données non initialisée')
    return
  }
  try {
    const result = await storage.value.allDocs({ include_docs: true })
    const toDelete = result.rows
      .map((row) => row.doc)
      .filter((doc) => !!doc && !doc._id?.startsWith('_'))
      .map((doc) => ({
        ...doc,
        _deleted: true,
      }))
    if (toDelete.length) {
      await storage.value.bulkDocs(toDelete)
      fetchData()
      console.log('✅ Tous les documents utilisateur supprimés')
    } else {
      console.log('Aucun document à supprimer')
    }
  } catch (err) {
    console.error('Erreur suppression globale', err)
  }
}

// Premier commentaire pour un post (tout via Mango)
const fetchFirstCommentForPost = async (postId: string) => {
  if (!storage.value) return
  await (storage.value as any).createIndex({
    index: { fields: ['type', 'postId', 'created_at'] },
  })
  const result = await (storage.value as any).find({
    selector: { type: 'comment', postId },
    sort: [{ created_at: 'asc' }],
    limit: 1,
  })
  firstCommentsByPost.value[postId] = result.docs[0] || null
}

// Chargement des posts via db.find (aucun tri JS)
const fetchData = async () => {
  if (!storage.value) {
    console.warn('Base de données non initialisée')
    return
  }
  try {
    await (storage.value as any).createIndex({
      index: { fields: ['type', 'created_at'] },
    })
    const result = await (storage.value as any).find({
      selector: { type: 'post' },
      sort: [{ created_at: 'desc' }],
    })
    postsData.value = (result.docs as Post[]).filter((doc) => !doc._id?.startsWith('_'))
    console.log('✅ Données récupérées (Mango) :', postsData.value)

    firstCommentsByPost.value = {}
    for (const post of postsData.value) {
      if (post._id) {
        await fetchFirstCommentForPost(post._id)
      }
    }
  } catch (error) {
    console.error('❌ Erreur fetchData Mango :', error)
  }
}

const startEdit = (post: Post) => {
  editingPost.value = { ...post }
}
const cancelEdit = () => {
  editingPost.value = null
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
    likes: 0,
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
        likes: 0,
      }
    })
    .catch((error) => {
      console.error("❌ Erreur lors de l'ajout du document :", error)
    })
}

const like = async (post: Post) => {
  if (!storage.value || !post._id || !post._rev) return
  try {
    const nbLikes = typeof post.likes === 'number' ? post.likes + 1 : 1
    await storage.value.put({ ...post, likes: nbLikes })
    fetchData()
  } catch (error) {
    console.error('Erreur like:', error)
  }
}

const showComments = async (postId: string) => {
  currentPostId.value = postId
  await fetchCommentsFor(postId)
}

const fetchCommentsFor = async (postId: string) => {
  if (!storage.value) return
  await (storage.value as any).createIndex({
    index: { fields: ['type', 'postId', 'created_at'] },
  })
  const result = await (storage.value as any).find({
    selector: {
      type: 'comment',
      postId,
    },
    sort: [{ created_at: 'asc' }],
  })
  commentList.value = result.docs as Comment[]
}

const saveComment = async (postId: string) => {
  if (!storage.value) return
  try {
    if (commentEditing.value) {
      const doc = await storage.value.get(commentEditing.value._id)
      await storage.value.put({ ...doc, content: newCommentContent.value })
      commentEditing.value = null
      newCommentContent.value = ''
    } else {
      const comment: Comment = {
        type: 'comment',
        postId,
        author: commentAuthor.value || 'Anonyme',
        content: newCommentContent.value,
        created_at: new Date().toISOString(),
      }
      await storage.value.post(comment)
      newCommentContent.value = ''
      commentAuthor.value = ''
    }
    await fetchCommentsFor(postId)
    fetchData()
  } catch (error) {
    console.error('Erreur saveComment:', error)
  }
}

const deleteComment = async (_id: string, _rev: string) => {
  if (!storage.value || !currentPostId.value) return
  try {
    await storage.value.remove(_id, _rev)
    await fetchCommentsFor(currentPostId.value)
    fetchData()
  } catch (error) {
    console.error('Erreur deleteComment:', error)
  }
}

const startEditComment = (comment: Comment) => {
  commentEditing.value = { _id: comment._id!, content: comment.content }
  newCommentContent.value = comment.content
}

const searchPosts = async () => {
  if (!storage.value) return
  const keyword = searchQuery.value.trim()
  if (!keyword) return fetchData()
  await storage.value.createIndex({ index: { fields: ['type', 'content'] } })
  const result = await (storage.value as any).find({
    selector: {
      type: 'post',
      content: { $regex: keyword },
    },
  })
  postsData.value = result.docs as Post[]
}

// Top 10 plus likés: tri & limit via Mango (aucun sort JS)
const sortByLikes = async () => {
  if (!storage.value) return
  await (storage.value as any).createIndex({
    index: { fields: ['type', 'likes'] },
  })
  const result = await (storage.value as any).find({
    selector: { type: 'post', likes: { $gte: 0 } },
    sort: [{ likes: 'desc' }],
    limit: 10,
  })
  postsData.value = result.docs as Post[]
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
      alert('Synchronisation terminée avec succès!')
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
  <div class="container">
    <header>
      <h1>📦 PouchDB Manager</h1>
      <div class="mode-toggle">
        <button @click="toggleMode" :class="{ active: !isOnline }">
          <span v-if="isOnline">🌐 Online</span>
          <span v-else>📴 Offline</span>
        </button>
      </div>
    </header>

    <div class="search-section">
      <div class="search-bar">
        <input
          v-model="searchQuery"
          @keyup.enter="searchPosts"
          placeholder="🔍 Rechercher par contenu..."
        />
        <button @click="searchPosts" class="btn-primary">Rechercher</button>
        <button @click="sortByLikes" class="btn-secondary">Top 10 par likes</button>
      </div>

      <div class="search-bar">
        <input
          v-model="searchFirstName"
          @input="searchByFirstName"
          placeholder="🔍 Rechercher par prénom..."
        />
        <button
          v-if="searchFirstName"
          @click="
            () => {
              searchFirstName = ''
              fetchData()
            }
          "
          class="btn-clear"
        >
          ✕
        </button>
      </div>
    </div>

    <div class="actions">
      <div class="factory-section">
        <input
          type="number"
          v-model.number="factoryCount"
          min="1"
          max="999"
          class="factory-input"
        />
        <button @click="populateFactory(factoryCount)" :disabled="isPopulating" class="btn-primary">
          {{ isPopulating ? 'Génération...' : `Générer ${factoryCount} docs` }}
        </button>
      </div>
      <button @click="syncDatabases" class="btn-sync">🔄 Synchroniser</button>
      <button @click="deleteAllDocuments" class="btn-danger">🗑️ Tout supprimer</button>
    </div>

    <form @submit.prevent="addDocument" class="add-form">
      <h3>➕ Nouveau document</h3>
      <div class="form-grid">
        <div class="form-group">
          <label>Prénom</label>
          <input v-model="newPost.name.first" required />
        </div>
        <div class="form-group">
          <label>Nom</label>
          <input v-model="newPost.name.last" />
        </div>
      </div>
      <div class="form-group">
        <label>Email</label>
        <input v-model="newPost.email" type="email" required />
      </div>
      <div class="form-group">
        <label>Contenu</label>
        <textarea
          v-model="newPost.content"
          rows="3"
          placeholder="Entrez votre contenu..."
        ></textarea>
      </div>
      <button type="submit" class="btn-primary">Ajouter</button>
    </form>

    <div class="posts">
      <article v-for="post in postsData" :key="post._id" class="post-card">
        <div v-if="!editingPost || editingPost._id !== post._id">
          <div class="post-header">
            <h2>{{ post.name.first }} {{ post.name.last }}</h2>
            <span class="post-date">
              {{ new Date(post.created_at).toLocaleDateString() }}
            </span>
          </div>
          <p class="post-email">{{ post.email }}</p>
          <p class="post-tags">{{ post.tags.join(', ') }}</p>
          <p v-if="post.content" class="post-content">{{ post.content }}</p>

          <!-- Premier commentaire (via Mango) -->
          <p v-if="firstCommentsByPost[post._id!]">
            <strong>1er commentaire :</strong>
            {{ firstCommentsByPost[post._id!]?.author }} –
            {{ firstCommentsByPost[post._id!]?.content }}
          </p>

          <div class="post-actions">
            <button @click="startEdit(post)" class="btn-edit">✏️ Modifier</button>
            <button @click="deleteDocument(post._id!, post._rev!)" class="btn-delete">
              🗑️ Supprimer
            </button>
            <button @click="like(post)" class="btn-like">👍 {{ post.likes || 0 }}</button>
            <button
              v-if="currentPostId !== post._id"
              @click="showComments(post._id!)"
              class="btn-comment"
            >
              💬 Voir tous les commentaires
            </button>
          </div>
        </div>

        <div v-else class="edit-mode">
          <h3>✏️ Modifier le document</h3>
          <div class="form-grid">
            <div class="form-group">
              <label>Prénom</label>
              <input v-model="editingPost.name.first" required />
            </div>
            <div class="form-group">
              <label>Nom</label>
              <input v-model="editingPost.name.last" />
            </div>
          </div>
          <div class="form-group">
            <label>Email</label>
            <input v-model="editingPost.email" type="email" required />
          </div>
          <div class="form-group">
            <label>Contenu</label>
            <textarea v-model="editingPost.content" rows="3"></textarea>
          </div>
          <div class="edit-actions">
            <button @click="updateDocument()" class="btn-save">✓ Enregistrer</button>
            <button @click="cancelEdit()" class="btn-cancel">✕ Annuler</button>
          </div>
        </div>

        <div v-if="currentPostId === post._id" class="comments-section">
          <div v-if="commentList.length" class="comments-list">
            <div v-for="comment in commentList" :key="comment._id" class="comment">
              <strong>{{ comment.author }}:</strong>
              <span>{{ comment.content }}</span>
              <div class="comment-actions">
                <button @click="startEditComment(comment)" class="btn-icon">✏️</button>
                <button @click="deleteComment(comment._id!, comment._rev!)" class="btn-icon">
                  🗑️
                </button>
              </div>
            </div>
          </div>
          <form @submit.prevent="saveComment(post._id!)" class="comment-form">
            <input v-model="commentAuthor" placeholder="Votre nom" class="comment-author" />
            <input
              v-model="newCommentContent"
              placeholder="Votre commentaire..."
              required
              class="comment-input"
            />
            <button type="submit" class="btn-primary">
              {{ commentEditing ? '✓' : '➤' }}
            </button>
            <button
              v-if="commentEditing"
              @click="
                () => {
                  commentEditing = null
                  newCommentContent = ''
                }
              "
              type="button"
              class="btn-cancel"
            >
              ✕
            </button>
          </form>
        </div>
      </article>
    </div>
  </div>
</template>

<style scoped>
* {
  box-sizing: border-box;
}

body {
  margin: 0;
  padding: 0;
  font-family:
    -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
  background: #f5f7fa;
}

.container {
  max-width: 900px;
  margin: 0 auto;
  padding: 2rem 1rem;
}

header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 2rem;
  padding-bottom: 1rem;
  border-bottom: 2px solid #e2e8f0;
}

h1 {
  font-size: 2rem;
  color: #1a202c;
  margin: 0;
  font-weight: 700;
}

.mode-toggle button {
  padding: 0.5rem 1.25rem;
  border: 2px solid #4299e1;
  background: white;
  color: #4299e1;
  border-radius: 8px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
}

.mode-toggle button:hover {
  background: #4299e1;
  color: white;
}

.mode-toggle button.active {
  background: #ed8936;
  border-color: #ed8936;
  color: white;
}

.search-section {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  margin-bottom: 1.5rem;
}

.search-bar {
  display: flex;
  gap: 0.75rem;
  flex-wrap: wrap;
}

.search-bar input {
  flex: 1;
  min-width: 200px;
  padding: 0.625rem 1rem;
  border: 2px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.95rem;
}

.search-bar input:focus {
  outline: none;
  border-color: #4299e1;
}

.actions {
  display: flex;
  gap: 0.75rem;
  margin-bottom: 2rem;
  flex-wrap: wrap;
}

.factory-section {
  display: flex;
  gap: 0.5rem;
  align-items: center;
}

.factory-input {
  width: 70px;
  padding: 0.625rem;
  border: 2px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.95rem;
}

button {
  padding: 0.625rem 1.25rem;
  border: none;
  border-radius: 8px;
  font-size: 0.95rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
}

.btn-primary {
  background: #4299e1;
  color: white;
}

.btn-primary:hover:not(:disabled) {
  background: #3182ce;
}

.btn-primary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.btn-secondary {
  background: #805ad5;
  color: white;
}

.btn-secondary:hover {
  background: #6b46c1;
}

.btn-sync {
  background: #48bb78;
  color: white;
}

.btn-sync:hover {
  background: #38a169;
}

.btn-danger {
  background: #f56565;
  color: white;
}

.btn-danger:hover {
  background: #e53e3e;
}

.btn-clear {
  background: #cbd5e0;
  color: #2d3748;
  padding: 0.625rem;
  width: 40px;
}

.btn-clear:hover {
  background: #a0aec0;
}

.add-form {
  background: white;
  border-radius: 12px;
  padding: 1.5rem;
  margin-bottom: 2rem;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

.add-form h3 {
  margin: 0 0 1.25rem 0;
  color: #2d3748;
  font-size: 1.25rem;
}

.form-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 1rem;
}

.form-group {
  margin-bottom: 1rem;
}

.form-group label {
  display: block;
  margin-bottom: 0.5rem;
  color: #4a5568;
  font-weight: 600;
  font-size: 0.9rem;
}

.form-group input,
.form-group textarea {
  width: 100%;
  padding: 0.625rem;
  border: 2px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.95rem;
  font-family: inherit;
}

.form-group input:focus,
.form-group textarea:focus {
  outline: none;
  border-color: #4299e1;
}

.posts {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.post-card {
  background: white;
  border-radius: 12px;
  padding: 1.5rem;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
}

.post-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.75rem;
}

.post-header h2 {
  margin: 0;
  color: #2d3748;
  font-size: 1.25rem;
}

.post-date {
  color: #718096;
  font-size: 0.85rem;
}

.post-email {
  color: #4a5568;
  margin: 0.25rem 0;
  font-size: 0.95rem;
}

.post-tags {
  color: #805ad5;
  font-size: 0.85rem;
  margin: 0.25rem 0;
}

.post-content {
  color: #2d3748;
  margin: 1rem 0;
  line-height: 1.6;
}

.post-actions {
  display: flex;
  gap: 0.5rem;
  margin-top: 1rem;
  flex-wrap: wrap;
}

.btn-edit {
  background: #ed8936;
  color: white;
  padding: 0.5rem 1rem;
  font-size: 0.9rem;
}

.btn-edit:hover {
  background: #dd6b20;
}

.btn-delete {
  background: #f56565;
  color: white;
  padding: 0.5rem 1rem;
  font-size: 0.9rem;
}

.btn-delete:hover {
  background: #e53e3e;
}

.btn-like {
  background: #48bb78;
  color: white;
  padding: 0.5rem 1rem;
  font-size: 0.9rem;
}

.btn-like:hover {
  background: #38a169;
}

.btn-comment {
  background: #4299e1;
  color: white;
  padding: 0.5rem 1rem;
  font-size: 0.9rem;
}

.btn-comment:hover {
  background: #3182ce;
}

.edit-mode {
  background: #ebf8ff;
  border: 2px solid #4299e1;
  border-radius: 8px;
  padding: 1.25rem;
}

.edit-mode h3 {
  margin: 0 0 1rem 0;
  color: #2c5282;
}

.edit-actions {
  display: flex;
  gap: 0.5rem;
  margin-top: 1rem;
}

.btn-save {
  background: #48bb78;
  color: white;
  flex: 1;
}

.btn-save:hover {
  background: #38a169;
}

.btn-cancel {
  background: #cbd5e0;
  color: #2d3748;
  flex: 1;
}

.btn-cancel:hover {
  background: #a0aec0;
}

.comments-section {
  margin-top: 1rem;
  padding-top: 1rem;
  border-top: 2px solid #e2e8f0;
}

.comments-list {
  margin-bottom: 1rem;
}

.comment {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.5rem;
  background: #f7fafc;
  border-radius: 6px;
  margin-bottom: 0.5rem;
}

.comment strong {
  color: #2d3748;
}

.comment span {
  flex: 1;
  color: #000;
}

.comment-actions {
  display: flex;
  gap: 0.25rem;
}

.btn-icon {
  padding: 0.25rem 0.5rem;
  background: transparent;
  border: none;
  cursor: pointer;
  font-size: 0.9rem;
}

.btn-icon:hover {
  background: #e2e8f0;
  border-radius: 4px;
}

.comment-form {
  display: flex;
  gap: 0.5rem;
  align-items: center;
}

.comment-author {
  width: 120px;
  padding: 0.5rem;
  border: 2px solid #e2e8f0;
  border-radius: 6px;
  font-size: 0.9rem;
}

.comment-input {
  flex: 1;
  padding: 0.5rem;
  border: 2px solid #e2e8f0;
  border-radius: 6px;
  font-size: 0.9rem;
}

.comment-author:focus,
.comment-input:focus {
  outline: none;
  border-color: #4299e1;
}

.comment-form button {
  padding: 0.5rem 1rem;
  font-size: 0.9rem;
}

@media (max-width: 768px) {
  .container {
    padding: 1rem 0.5rem;
  }

  h1 {
    font-size: 1.5rem;
  }

  header {
    flex-direction: column;
    gap: 1rem;
    align-items: flex-start;
  }

  .search-bar,
  .actions {
    flex-direction: column;
  }

  .search-bar input {
    min-width: 100%;
  }

  .factory-section {
    width: 100%;
  }

  .form-grid {
    grid-template-columns: 1fr;
  }

  .post-actions {
    flex-direction: column;
  }

  .post-actions button {
    width: 100%;
  }

  .comment-form {
    flex-direction: column;
  }

  .comment-author,
  .comment-input {
    width: 100%;
  }

  .edit-actions {
    flex-direction: column;
  }
}

.post-card p {
  color: #000;
}
</style>
