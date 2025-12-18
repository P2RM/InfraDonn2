<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'

PouchDB.plugin(PouchDBFind)

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

const localPostsDB = ref<PouchDB.Database | null>(null)
const remotePostsDB = ref<PouchDB.Database | null>(null)
const postsStorage = ref<PouchDB.Database | null>(null)

const localCommentsDB = ref<PouchDB.Database | null>(null)
const remoteCommentsDB = ref<PouchDB.Database | null>(null)
const commentsStorage = ref<PouchDB.Database | null>(null)

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

const attachmentFile = ref<File | null>(null)
const attachmentPreview = ref<string | null>(null)
const attachmentPreviewByPost = ref<Record<string, string | null>>({})
const attachmentUploading = ref(false)
const attachmentPreviewType = ref<string | null>(null)
const attachmentPreviewByPostType = ref<Record<string, string | null>>({})
const inlinePreviewType = ref<string | null>(null)

const handleFileChange = (e: Event) => {
  const input = e.target as HTMLInputElement
  const file = input.files && input.files[0] ? input.files[0] : null
  if (!file) {
    clearSelectedAttachment()
    return
  }

  attachmentFile.value = file
  if (attachmentPreview.value) URL.revokeObjectURL(attachmentPreview.value)
  attachmentPreview.value = URL.createObjectURL(file)
  attachmentPreviewType.value = file.type || null
}

const clearSelectedAttachment = () => {
  if (attachmentPreview.value) {
    URL.revokeObjectURL(attachmentPreview.value)
    attachmentPreview.value = null
  }
  attachmentFile.value = null
  attachmentPreviewType.value = null
}

const fetchAttachmentPreviewFor = async (postId: string, attachmentName: string) => {
  if (!postsStorage.value) return null
  try {
    const blob: Blob = await (postsStorage.value as any).getAttachment(postId, attachmentName)
    if (!blob) return null

    if (attachmentPreviewByPost.value[postId]) URL.revokeObjectURL(attachmentPreviewByPost.value[postId]!)
    const url = URL.createObjectURL(blob)
    attachmentPreviewByPost.value[postId] = url
    attachmentPreviewByPostType.value[postId] = blob.type || null
    return { url, type: blob.type || null }
  } catch (err) {
    console.error('Erreur récupération attachment', err)
    return null
  }
}

const showAttachment = async (post: Post) => {
  if (!post._id) return
  const att: any = (post as any).attachment
  if (!att?.name) return

  const res = await fetchAttachmentPreviewFor(post._id, att.name)
  const type = res?.type || att.content_type || null

  if (res?.url) openInlinePreview(res.url, type)
  else alert('Impossible de récupérer la pièce jointe')
}

const showInlinePreview = ref(false)
const inlinePreviewUrl = ref<string | null>(null)
const openInlinePreview = (url: string, type: string | null) => {
  inlinePreviewUrl.value = url
  inlinePreviewType.value = type
  showInlinePreview.value = true
}

const closeInlinePreview = () => {
  inlinePreviewUrl.value = null
  inlinePreviewType.value = null
  showInlinePreview.value = false
}


const removeAttachment = async (postId: string, attachmentName: string) => {
  if (!postsStorage.value) return
  try {
    const doc: any = await postsStorage.value.get(postId)
    await (postsStorage.value as any).removeAttachment(postId, attachmentName, doc._rev)
    const doc2: any = await postsStorage.value.get(postId)
    if (doc2.attachment) delete doc2.attachment
    await postsStorage.value.put(doc2)
    await fetchData()
    alert('Pièce jointe supprimée')
  } catch (err) {
    console.error('Erreur suppression attachment', err)
    alert('Impossible de supprimer la pièce jointe')
  }
}

const removeAttachmentForEditing = async () => {
  if (!editingPost.value || !editingPost.value._id) return
  const att: any = (editingPost.value as any).attachment
  if (!att || !att.name) return
  if (!confirm('Supprimer cette pièce jointe ?')) return
  try {
    await removeAttachment(editingPost.value._id, att.name)
    const doc: any = await postsStorage.value!.get(editingPost.value._id)
    editingPost.value._rev = doc._rev
    clearSelectedAttachment()
  } catch (err) {
    console.error('Erreur removal editing', err)
  }
}

const firstCommentsByPost = ref<Record<string, Comment | null>>({})

const ensureIndexes = async () => {
  try {
    if (postsStorage.value) {
      const db: any = postsStorage.value
      await db.createIndex({ index: { fields: ['type', 'created_at'] } })
      await db.createIndex({ index: { fields: ['type', 'likes'] } })
      await db.createIndex({ index: { fields: ['name.first'] } })
      await db.createIndex({ index: { fields: ['type', 'content'] } })
    }

    if (commentsStorage.value) {
      const db: any = commentsStorage.value
      await db.createIndex({ index: { fields: ['type', 'postId', 'created_at'] } })
    }
  } catch (e) {
    console.error('❌ Erreur index', e)
  }
}

const toggleMode = async () => {
  isOnline.value = !isOnline.value
  postsStorage.value = isOnline.value ? remotePostsDB.value : localPostsDB.value
  commentsStorage.value = isOnline.value ? remoteCommentsDB.value : localCommentsDB.value
  await ensureIndexes()
  fetchData()
}

const populateFactory = async (nb = 100) => {
  if (!postsStorage.value) return
  isPopulating.value = true
  try {
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
        likes: Math.floor(Math.random() * 20),
      })
    }
    console.log('👉 bulkDocs start', nb)
    const res = await postsStorage.value.bulkDocs(bulkDocs)
    console.log('✅ bulkDocs ok', res)
    await fetchData()
  } catch (err) {
    console.error('❌ bulkDocs error', err)
  } finally {
    isPopulating.value = false
  }
}

const searchByFirstName = async () => {
  if (!postsStorage.value) return
  await ensureIndexes()
  const filter = searchFirstName.value.trim()
  if (!filter) {
    fetchData()
    return
  }
  try {
    const result = await (postsStorage.value as any).find({
      selector: {
        type: 'post',
        'name.first': { $regex: `^${filter}` },
      },
    })
    postsData.value = result.docs as Post[]
  } catch (e) {
    console.error('Recherche impossible', e)
  }
}

const initDatabase = async () => {
  localPostsDB.value = new PouchDB('local_posts_db')
  localCommentsDB.value = new PouchDB('local_comments_db')

  remotePostsDB.value = new PouchDB('http://admin:admin@127.0.0.1:5984/posts_db')
  remoteCommentsDB.value = new PouchDB('http://admin:admin@127.0.0.1:5984/comments_db')

  postsStorage.value = isOnline.value ? remotePostsDB.value : localPostsDB.value
  commentsStorage.value = isOnline.value ? remoteCommentsDB.value : localCommentsDB.value

  await ensureIndexes()
  console.log('✅ Bases POSTS et COMMENTS initialisées')
}

const deleteAllDocuments = async () => {
  try {
    if (postsStorage.value) {
      const result = await postsStorage.value.allDocs({ include_docs: true })
      const toDelete = result.rows
        .map((row) => row.doc)
        .filter((doc) => !!doc && !doc._id?.startsWith('_'))
        .map((doc: any) => ({ ...doc, _deleted: true }))
      if (toDelete.length) await postsStorage.value.bulkDocs(toDelete)
    }

    if (commentsStorage.value) {
      const result = await commentsStorage.value.allDocs({ include_docs: true })
      const toDelete = result.rows
        .map((row) => row.doc)
        .filter((doc) => !!doc && !doc._id?.startsWith('_'))
        .map((doc: any) => ({ ...doc, _deleted: true }))
      if (toDelete.length) await commentsStorage.value.bulkDocs(toDelete)
    }

    fetchData()
  } catch (err) {
    console.error('Erreur suppression globale', err)
  }
}

const fetchFirstCommentForPost = async (postId: string) => {
  if (!commentsStorage.value) return
  await ensureIndexes()
  const result = await (commentsStorage.value as any).find({
    selector: { type: 'comment', postId },
    sort: [{ created_at: 'asc' }],
    limit: 1,
  })
  firstCommentsByPost.value[postId] = result.docs[0] || null
}

const fetchData = async () => {
  if (!postsStorage.value) {
    console.warn('Base POSTS non initialisée')
    return
  }
  try {
    await ensureIndexes()
    const result = await (postsStorage.value as any).find({
      selector: { type: 'post' },
      sort: [{ created_at: 'desc' }],
    })
    postsData.value = (result.docs as Post[]).filter((doc) => !doc._id?.startsWith('_'))
    console.log('✅ Données POSTS récupérées (Mango) :', postsData.value)

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
  editingPost.value = { ...post, name: { ...post.name } }
  const att: any = (post as any).attachment

  if (post._id && att?.name) {
    ;(async () => {
      const res = await fetchAttachmentPreviewFor(post._id!, att.name)
      if (res) {
        attachmentPreview.value = res.url
        attachmentPreviewType.value = res.type || att.content_type || null
      }
    })()
  } else {
    clearSelectedAttachment()
  }
}

const cancelEdit = () => {
  editingPost.value = null
}

const deleteDocument = (docId: string, docRev: string) => {
  if (!postsStorage.value) {
    console.warn('Base POSTS non initialisée')
    return
  }
  postsStorage.value
    .remove(docId, docRev)
    .then((response) => {
      console.log('✅ Document supprimé :', response)
      fetchData()
    })
    .catch((error) => {
      console.error('❌ Erreur lors de la suppression du document :', error)
    })
}

const updateDocument = async () => {
  if (!postsStorage.value) {
    console.warn('Base POSTS non initialisée')
    return
  }
  if (!editingPost.value || !editingPost.value._id) {
    console.warn('⚠️ Le document doit avoir un _id pour être mis à jour')
    return
  }
  try {
    const putRes: any = await postsStorage.value.put(editingPost.value)
    let rev = putRes.rev

    if (attachmentFile.value) {
      try {
        attachmentUploading.value = true
        const file = attachmentFile.value
        const putAtt: any = await (postsStorage.value as any).putAttachment(
          editingPost.value._id,
          file.name,
          rev,
          file,
          file.type
        )
        const doc: any = await postsStorage.value.get(editingPost.value._id)
        doc.attachment = { name: file.name, content_type: file.type }
        const updated: any = await postsStorage.value.put(doc)
        rev = updated.rev
        clearSelectedAttachment()
        console.log('✅ Attachment replaced:', putAtt)
      } catch (err) {
        console.error('❌ Erreur lors du remplacement de l attachment :', err)
      } finally {
        attachmentUploading.value = false
      }
    }

    await fetchData()
    editingPost.value = null
  } catch (error) {
    console.error('❌ Erreur lors de la mise à jour du document :', error)
  }
}

const addDocument = () => {
  if (!postsStorage.value) {
    console.warn('Base POSTS non initialisée')
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
  ;(async () => {
    try {
      const response: any = await postsStorage.value!.post(postToAdd)
      console.log('✅ Document ajouté :', response)

      if (attachmentFile.value) {
        try {
          attachmentUploading.value = true
          const file = attachmentFile.value
          const putRes: any = await (postsStorage.value as any).putAttachment(
            response.id,
            file.name,
            response.rev,
            file,
            file.type
          )

          const doc: any = await postsStorage.value.get(response.id)
          doc.attachment = { name: file.name, content_type: file.type }
          await postsStorage.value.put(doc)
          clearSelectedAttachment()
          console.log('✅ Attachment ajouté:', putRes)
        } catch (err) {
          console.error('❌ Erreur lors de l upload attachment :', err)
        } finally {
          attachmentUploading.value = false
        }
      }

      await fetchData()
      newPost.value = {
        type: 'post',
        name: { first: '', last: '' },
        email: '',
        tags: [],
        created_at: '',
        content: '',
        likes: 0,
      }
    } catch (error) {
      console.error("❌ Erreur lors de l'ajout du document :", error)
    }
  })()
}

const like = async (post: Post) => {
  if (!postsStorage.value || !post._id || !post._rev) return
  try {
    const nbLikes = typeof post.likes === 'number' ? post.likes + 1 : 1
    await postsStorage.value.put({ ...post, likes: nbLikes })
    fetchData()
  } catch (error) {
    console.error('Erreur like:', error)
  }
}

const showComments = async (postId: string) => {
  currentPostId.value = postId
  await fetchCommentsFor(postId)
}

const createTestDocWithAttachment = async () => {
  if (!postsStorage.value) return
  try {
    const doc: any = {
      type: 'post',
      name: { first: 'Test', last: 'Attachment' },
      email: 'test@local',
      tags: ['test', 'attachment'],
      created_at: new Date().toISOString(),
      content: 'Doc de test avec attachment',
      likes: 0,
    }
    const res: any = await postsStorage.value.post(doc)
    const blob = new Blob(['Bonjour, ceci est un test d attachment'], { type: 'text/plain' })
    await (postsStorage.value as any).putAttachment(res.id, 'hello.txt', res.rev, blob, 'text/plain')
    const doc2: any = await postsStorage.value.get(res.id)
    doc2.attachment = { name: 'hello.txt', content_type: 'text/plain' }
    await postsStorage.value.put(doc2)
    await fetchData()
    alert('Document de test créé')
  } catch (err) {
    console.error('Erreur création test attachment', err)
    alert('Erreur création test')
  }
}

const fetchCommentsFor = async (postId: string) => {
  if (!commentsStorage.value) return
  await ensureIndexes()
  const result = await (commentsStorage.value as any).find({
    selector: {
      type: 'comment',
      postId,
    },
    sort: [{ created_at: 'asc' }],
  })
  commentList.value = result.docs as Comment[]
}

const saveComment = async (postId: string) => {
  if (!commentsStorage.value) return
  try {
    if (commentEditing.value) {
      const doc = await commentsStorage.value.get(commentEditing.value._id)
      await commentsStorage.value.put({ ...doc, content: newCommentContent.value })
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
      await commentsStorage.value.post(comment)
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
  if (!commentsStorage.value || !currentPostId.value) return
  try {
    await commentsStorage.value.remove(_id, _rev)
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
  if (!postsStorage.value) return
  const keyword = searchQuery.value.trim()
  if (!keyword) return fetchData()
  await ensureIndexes()
  const result = await (postsStorage.value as any).find({
    selector: {
      type: 'post',
      content: { $regex: keyword },
    },
  })
  postsData.value = result.docs as Post[]
}

const sortByLikes = async () => {
  if (!postsStorage.value) return
  await ensureIndexes()
  const result = await (postsStorage.value as any).find({
    selector: { type: 'post', likes: { $gte: 0 } },
    sort: [{ likes: 'desc' }],
    limit: 10,
  })
  postsData.value = result.docs as Post[]
}

const syncDatabases = async () => {
  if (!localPostsDB.value || !remotePostsDB.value || !localCommentsDB.value || !remoteCommentsDB.value) {
    alert('Bases non initialisées')
    return
  }
  try {
    await localPostsDB.value.sync(remotePostsDB.value, { live: false, retry: false })
    await localCommentsDB.value.sync(remoteCommentsDB.value, { live: false, retry: false })
    fetchData()
    alert('Synchronisation terminée avec succès!')
  } catch (err) {
    console.error('❌ Erreur lors de la synchronisation:', err)
    alert('Erreur lors de la synchronisation')
  }
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
      <button @click="createTestDocWithAttachment" class="btn-secondary">🧪 Test att.</button>
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
          <div class="form-group">
            <label>Pièce jointe (optionnelle)</label>
            <input type="file" @change="handleFileChange" />
            <div v-if="attachmentPreview" style="margin-top:8px">
              <strong>Aperçu:</strong>
              <div style="margin-top:6px">
                <img v-if="attachmentPreviewType && attachmentPreviewType.startsWith('image/')" :src="attachmentPreview" alt="preview" style="max-width:200px; max-height:150px;" />
                <a v-else :href="attachmentPreview" target="_blank">Ouvrir la pièce jointe</a>
              </div>
            </div>
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

          <div v-if="(post as any).attachment && (post as any).attachment.name" style="margin-top:8px">
              <button @click="showAttachment(post)" class="btn-secondary">📎 Voir la pièce jointe</button>
              <button
                @click="() => { if(confirm('Supprimer la pièce jointe ?')) removeAttachment(post._id!, (post as any).attachment.name) }"
                class="btn-delete"
                style="margin-left:8px"
              >
                🗑️ Suppr. PJ
              </button>
          </div>

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
  <div class="form-group">
    <label>Pièce jointe (optionnelle)</label>
    <input type="file" @change="handleFileChange" />

    <div v-if="attachmentPreview" style="margin-top:8px">
      <strong>Aperçu:</strong>
      <div style="margin-top:6px">
        <img
          v-if="attachmentPreviewType && attachmentPreviewType.startsWith('image/')"
          :src="attachmentPreview"
          alt="preview"
          style="max-width:200px; max-height:150px;"
        />
        <a v-else :href="attachmentPreview" target="_blank">
          Ouvrir la pièce jointe
        </a>
      </div>
    </div>
  </div>

  <div class="edit-actions">
    <button @click="updateDocument()" class="btn-save">✓ Enregistrer</button>
    <button @click="cancelEdit()" class="btn-cancel">✕ Annuler</button>
  </div>
</div>

          <div class="form-group">
  <label>Pièce jointe (optionnelle)</label>
  <input type="file" @change="handleFileChange" />

  <div v-if="attachmentPreview" style="margin-top:8px">
    <strong>Aperçu:</strong>
    <div style="margin-top:6px">
      <img
        v-if="attachmentPreviewType && attachmentPreviewType.startsWith('image/')"
        :src="attachmentPreview"
        alt="preview"
        style="max-width:200px; max-height:150px;"
      />
      <a v-else :href="attachmentPreview" target="_blank">Ouvrir la pièce jointe</a>
    </div>
  </div>
</div>
            <div class="form-group">
              <label>Pièce jointe (optionnelle)</label>
              <input type="file" @change="handleFileChange" />
                <div v-if="attachmentPreview" style="margin-top:8px">
                <strong>Aperçu:</strong>
                <div style="margin-top:6px">
                  <img v-if="attachmentPreviewType && attachmentPreviewType.startsWith('image/')" :src="attachmentPreview" alt="preview" style="max-width:200px; max-height:150px;" />
                  <a v-else :href="attachmentPreview" target="_blank">Ouvrir la pièce jointe</a>
                </div>
              </div>
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
  <div v-if="showInlinePreview" class="inline-preview-backdrop" @click.self="closeInlinePreview">
    <div class="inline-preview">
      <button class="btn-cancel" style="position:absolute;right:8px;top:8px" @click="closeInlinePreview">✕</button>
      <div style="padding:12px;">
        <img v-if="inlinePreviewType && inlinePreviewType.startsWith('image/')" :src="inlinePreviewUrl" alt="preview" style="max-width:90vw; max-height:80vh;" />
        <div v-else>
          <a :href="inlinePreviewUrl" target="_blank">Ouvrir la pièce jointe</a>
        </div>
      </div>
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

.inline-preview-backdrop {
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,0.45);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 9999;
}
.inline-preview {
  background: white;
  border-radius: 8px;
  max-width: 95vw;
  max-height: 90vh;
  position: relative;
}
</style>
