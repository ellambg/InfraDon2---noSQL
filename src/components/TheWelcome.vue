<script setup lang="ts">
import { onMounted, ref, watch } from 'vue'
import PouchDB from 'pouchdb'
import PouchFind from 'pouchdb-find'

PouchDB.plugin(PouchFind)

// types
interface Comment {
  id: string
  author: string
  text: string
  created_at: string
}

interface Post {
  _id?: string
  _rev?: string
  post_name: string
  post_content: string
  attributes: string[]
  likes: number
  comments: Comment[]
}

// state
const remoteDb = ref<PouchDB.Database | null>(null)
const localDb = ref<PouchDB.Database | null>(null)

const postsData = ref<Post[]>([])

const newPost = ref<Post>({
  post_name: '',
  post_content: '',
  attributes: [],
  likes: 0,
  comments: [],
})

const editingPost = ref<Post | null>(null)

const searchTerm = ref('')
const sortByLikes = ref(false)
const isOnline = ref(true)

const commentingPostId = ref<string | null>(null)
const newCommentText = ref('')

// édition de commentaire
const editingComment = ref<{
  postId: string
  commentId: string
  text: string
} | null>(null)

// visibilité des commentaires par post
const commentsVisibility = ref<Record<string, boolean>>({})

const toggleCommentsVisibility = (postId: string) => {
  commentsVisibility.value[postId] = !commentsVisibility.value[postId]
}

// init db
const initDatabases = async () => {
  const remote = new PouchDB('http://admin:Infradon2_25!@localhost:5984/infradon2_db1')
  const local = new PouchDB('infradon2_local')

  remoteDb.value = remote
  localDb.value = local

  await createIndexes()
  await syncFromRemote()
}

// index
const createIndexes = async () => {
  if (!localDb.value) return

  await localDb.value.createIndex({
    index: { fields: ['post_name'] },
  })

  await localDb.value.createIndex({
    index: { fields: ['likes'] },
  })
}

// read all (local)
const fetchData = async () => {
  if (!localDb.value) return

  try {
    const result = await localDb.value.allDocs({ include_docs: true })
    postsData.value = result.rows
      .map((row: any) => row.doc as Post)
      .filter((doc: unknown): doc is Post => !!doc)
  } catch (error) {
    console.error('Erreur fetchData :', error)
  }
}

// afficher les 10 documents les plus likés (côté DB)
const fetchTop10Liked = async () => {
  if (!localDb.value) return

  try {
    const result = await localDb.value.find({
      selector: { likes: { $gte: 0 } },
      sort: [{ likes: 'desc' }],
      limit: 10,
    })
    postsData.value = result.docs as Post[]
  } catch (error) {
    console.error('Erreur fetchTop10Liked :', error)
  }
}

// réplication serveur -> local
const syncFromRemote = async () => {
  if (!remoteDb.value || !localDb.value) return

  try {
    await PouchDB.replicate(remoteDb.value, localDb.value)
    await fetchData()
  } catch (error) {
    console.error('Erreur syncFromRemote :', error)
  }
}

// réplication local -> serveur
const syncToRemote = async () => {
  if (!remoteDb.value || !localDb.value) return

  try {
    await PouchDB.replicate(localDb.value, remoteDb.value)
  } catch (error) {
    console.error('Erreur syncToRemote :', error)
  }
}

// sync complète (bouton)
const fullSync = async () => {
  await syncToRemote()
  await syncFromRemote()
}

// create
const addPost = async () => {
  if (!localDb.value) return

  const doc: Post = {
    _id: new Date().toISOString(),
    post_name: newPost.value.post_name.trim(),
    post_content: newPost.value.post_content.trim(),
    attributes: (newPost.value.attributes || []).map((a) => String(a).trim()).filter(Boolean),
    likes: 0,
    comments: [],
  }

  if (!doc.post_name || !doc.post_content) {
    console.warn('Champs requis manquants')
    return
  }

  try {
    await localDb.value.put(doc)
    newPost.value.post_name = ''
    newPost.value.post_content = ''
    newPost.value.attributes = []
    await fetchData()
  } catch (error) {
    console.error('Erreur addPost :', error)
  }
}

// update post
const startEdit = (post: Post) => {
  editingPost.value = JSON.parse(JSON.stringify(post))
}

const cancelEdit = () => {
  editingPost.value = null
}

const updatePost = async () => {
  if (!localDb.value) return
  const doc = editingPost.value
  if (!doc || !doc._id || !doc._rev) {
    console.warn('_id et _rev requis')
    return
  }

  doc.post_name = doc.post_name.trim()
  doc.post_content = doc.post_content.trim()
  doc.attributes = (doc.attributes || []).map((a) => String(a).trim()).filter(Boolean)

  try {
    await localDb.value.put(doc)
    editingPost.value = null
    await fetchData()
  } catch (error) {
    console.error('Erreur updatePost :', error)
  }
}

// delete post
const deletePost = async (docId?: string, docRev?: string) => {
  if (!localDb.value || !docId || !docRev) return

  try {
    await localDb.value.remove(docId, docRev)
    await fetchData()
  } catch (error) {
    console.error('Erreur deletePost :', error)
  }
}

// likes
const likePost = async (post: Post) => {
  if (!localDb.value || !post._id) return
  try {
    const current = (await localDb.value.get(post._id)) as Post
    current.likes = (current.likes || 0) + 1
    await localDb.value.put(current)
    await fetchData()
  } catch (error) {
    console.error('Erreur likePost :', error)
  }
}

// commentaires
const startComment = (postId: string) => {
  commentingPostId.value = postId
  newCommentText.value = ''
  editingComment.value = null
}

const addComment = async (post: Post) => {
  if (!localDb.value || !post._id) return
  if (!newCommentText.value.trim()) return

  try {
    const current = (await localDb.value.get(post._id)) as Post
    const comments = current.comments || []

    comments.push({
      id: new Date().toISOString(),
      author: 'Anonyme',
      text: newCommentText.value.trim(),
      created_at: new Date().toISOString(),
    })

    current.comments = comments
    await localDb.value.put(current)
    commentingPostId.value = null
    newCommentText.value = ''
    await fetchData()
  } catch (error) {
    console.error('Erreur addComment :', error)
  }
}

const deleteComment = async (post: Post, commentId: string) => {
  if (!localDb.value || !post._id) return

  try {
    const current = (await localDb.value.get(post._id)) as Post
    current.comments = (current.comments || []).filter((c) => c.id !== commentId)
    await localDb.value.put(current)
    await fetchData()
  } catch (error) {
    console.error('Erreur deleteComment :', error)
  }
}

// édition de commentaire
const startEditComment = (post: Post, comment: Comment) => {
  if (!post._id) return
  editingComment.value = {
    postId: post._id,
    commentId: comment.id,
    text: comment.text,
  }
  commentingPostId.value = null
}

const cancelEditComment = () => {
  editingComment.value = null
}

const updateComment = async (post: Post) => {
  if (!localDb.value || !post._id || !editingComment.value) return

  try {
    const current = (await localDb.value.get(post._id)) as Post
    current.comments = (current.comments || []).map((c) =>
      c.id === editingComment.value!.commentId
        ? { ...c, text: editingComment.value!.text.trim() }
        : c,
    )
    await localDb.value.put(current)
    editingComment.value = null
    await fetchData()
  } catch (error) {
    console.error('Erreur updateComment :', error)
  }
}

// recherche + tri avec find()
const runQuery = async () => {
  if (!localDb.value) return

  const term = searchTerm.value.trim()
  const selector: any = {}

  if (term) {
    selector.post_name = { $regex: term }
  }

  let sort: any[] | undefined
  if (sortByLikes.value) {
    selector.likes = { $gte: 0 }
    sort = [{ likes: 'desc' }]
  }

  try {
    const query: any = {
      selector: Object.keys(selector).length ? selector : { _id: { $gte: null } },
    }

    if (sort) {
      query.sort = sort
    }

    const result = await localDb.value.find(query)
    postsData.value = result.docs as Post[]
  } catch (error) {
    console.error('Erreur runQuery :', error)
  }
}

// factory
const createFakePost = (i: number): Post => ({
  post_name: `Message ${i}`,
  post_content: `Contenu du message ${i}`,
  attributes: ['demo', i % 2 === 0 ? 'pair' : 'impair'],
  likes: Math.floor(Math.random() * 20),
  comments: [],
})

const addManyFakePosts = async (count = 20) => {
  if (!localDb.value) return
  const docs: any[] = []

  for (let i = 0; i < count; i++) {
    const now = new Date().toISOString() + `-${i}`
    docs.push({
      _id: now,
      ...createFakePost(i),
    })
  }

  try {
    await localDb.value.bulkDocs(docs)
    await fetchData()
  } catch (error) {
    console.error('Erreur addManyFakePosts :', error)
  }
}

// watchers
watch(isOnline, async (value) => {
  if (value) {
    await fullSync()
  }
})

watch([searchTerm, sortByLikes], () => {
  runQuery()
})

// lifecycle
onMounted(async () => {
  await initDatabases()
})
</script>

<template>
  <div class="container">
    <h1>InfraDon - CouchDB + Vue 3</h1>

    <div class="actions">
      <button @click="fetchData">Rafraîchir (local)</button>
      <button @click="fullSync" :disabled="!isOnline">Synchroniser avec le serveur</button>

      <label class="inline-label">
        <input type="checkbox" v-model="isOnline" />
        Mode online
      </label>

      <button @click="addManyFakePosts(20)">Générer 20 faux messages</button>
      <button @click="fetchTop10Liked">Top 10 les plus likés</button>
    </div>

    <div class="search-bar">
      <input v-model="searchTerm" type="text" placeholder="Rechercher par nom" />
      <label class="inline-label">
        <input type="checkbox" v-model="sortByLikes" />
        Trier par likes
      </label>
    </div>

    <div class="form">
      <h2>Ajouter un document</h2>

      <input v-model="newPost.post_name" placeholder="Nom (post_name)" type="text" />
      <input
        v-model="newPost.post_content"
        placeholder="Contenu / Description (post_content)"
        type="text"
      />
      <input
        :value="(newPost.attributes || []).join(', ')"
        placeholder="Attributs séparés par des virgules"
        type="text"
        @input="newPost.attributes = ($event.target as HTMLInputElement).value.split(',')"
      />
      <button @click="addPost">Ajouter</button>
    </div>

    <hr />

    <div v-if="postsData.length === 0">
      <p>Aucune donnée trouvée.</p>
    </div>

    <article v-for="post in postsData" :key="post._id" class="item">
      <!-- vue normale -->
      <template v-if="!editingPost || editingPost._id !== post._id">
        <h2>{{ post.post_name }}</h2>
        <p>{{ post.post_content }}</p>
        <p>Attributs : {{ (post.attributes || []).join(', ') }}</p>
        <p>Likes : {{ post.likes || 0 }}</p>

        <div class="row">
          <button @click="likePost(post)">{{ post.likes > 0 ? '❤️' : '🤍' }} Like</button>

          <button @click="startEdit(post)">Modifier</button>
          <button @click="deletePost(post._id, post._rev)">Supprimer</button>
        </div>

        <div class="comments">
          <h4>Commentaires ({{ post.comments?.length || 0 }})</h4>

          <!-- Premier commentaire (une seule fois) -->
          <div v-if="post.comments?.length" class="first-comment">
            <p>
              <strong>Premier commentaire :</strong><br />
              <strong>{{ post.comments[0].author }}</strong> — {{ post.comments[0].text }}
            </p>

            <div class="row">
              <button @click="startEditComment(post, post.comments[0])">Modifier</button>
              <button @click="deleteComment(post, post.comments[0].id)">Supprimer</button>
            </div>
          </div>

          <!-- Aucun commentaire -->
          <div v-else>
            <p>Aucun commentaire pour l’instant.</p>
          </div>

          <!-- Bouton pour afficher les autres -->
          <div v-if="post.comments && post.comments.length > 1">
            <button class="toggle-btn" @click="toggleCommentsVisibility(post._id!)">
              {{
                commentsVisibility[post._id!]
                  ? 'Masquer les autres commentaires'
                  : 'Afficher les autres commentaires'
              }}
            </button>

            <!-- Les AUTRES commentaires (sauf le premier) -->
            <ul v-if="commentsVisibility[post._id!]">
              <li v-for="c in post.comments.slice(1)" :key="c.id">
                <template
                  v-if="
                    editingComment &&
                    editingComment.postId === post._id &&
                    editingComment.commentId === c.id
                  "
                >
                  <input v-model="editingComment.text" type="text" />
                  <div class="row">
                    <button @click="updateComment(post)">Enregistrer</button>
                    <button @click="cancelEditComment">Annuler</button>
                  </div>
                </template>

                <template v-else>
                  <span>
                    <strong>{{ c.author }}</strong> — {{ c.text }}
                  </span>
                  <div class="row">
                    <button @click="startEditComment(post, c)">Modifier</button>
                    <button @click="deleteComment(post, c.id)">Supprimer</button>
                  </div>
                </template>
              </li>
            </ul>
          </div>

          <!-- Formulaire d'ajout -->
          <div v-if="commentingPostId === post._id" class="comment-form">
            <input v-model="newCommentText" type="text" placeholder="Votre commentaire" />
            <button @click="addComment(post)">Envoyer</button>
          </div>
          <button v-else @click="startComment(post._id!)">Ajouter un commentaire</button>
        </div>
      </template>

      <!-- mode édition post -->
      <template v-else>
        <div class="editBox">
          <h3>Modifier le document</h3>
          <input v-model="editingPost.post_name" placeholder="Nom (post_name)" type="text" />
          <input
            v-model="editingPost.post_content"
            placeholder="Contenu / Description"
            type="text"
          />
          <input
            :value="(editingPost.attributes || []).join(', ')"
            placeholder="Attributs (séparés par des virgules)"
            type="text"
            @input="
              editingPost &&
              (editingPost.attributes = ($event.target as HTMLInputElement).value
                .split(',')
                .map((s) => s.trim()))
            "
          />
          <div class="row">
            <button @click="updatePost">Enregistrer</button>
            <button @click="cancelEdit">Annuler</button>
          </div>
        </div>
      </template>
    </article>
  </div>
</template>

<style scoped>
.container {
  min-height: 100vh;
  padding: 2rem 1.5rem;
  max-width: 960px;
  margin: 0 auto;
  color: #f5f5f5;
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
}

/* Barre d’actions en haut */
.actions {
  display: flex;
  flex-wrap: wrap; /* passe sur plusieurs lignes si petit écran */
  gap: 0.75rem;
  align-items: center;
}

.search-bar {
  display: flex;
  flex-wrap: wrap;
  gap: 0.75rem;
}

.inline-label {
  display: inline-flex;
  align-items: center;
  gap: 0.35rem;
  font-size: 0.9rem;
  opacity: 0.9;
}

/* Carte formulaire */
.form {
  background: linear-gradient(135deg, #1f2933, #111827);
  border-radius: 12px;
  padding: 1.5rem;
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
  box-shadow: 0 14px 30px rgba(0, 0, 0, 0.35);
}

.form h2 {
  margin: 0 0 0.5rem;
}

/* Inputs + boutons */
input {
  padding: 0.6rem 0.75rem;
  border-radius: 8px;
  border: 1px solid #374151;
  background: #111827;
  color: #f9fafb;
  font-size: 0.95rem;
  transition:
    border-color 0.15s ease,
    box-shadow 0.15s ease,
    background 0.15s ease;
}

input::placeholder {
  color: #6b7280;
}

input:focus {
  outline: none;
  border-color: #10b981;
  box-shadow: 0 0 0 1px rgba(16, 185, 129, 0.6);
  background: #020617;
}

button {
  background: linear-gradient(135deg, #10b981, #059669);
  color: #f9fafb;
  padding: 0.55rem 0.9rem;
  border: none;
  cursor: pointer;
  border-radius: 999px;
  font-size: 0.9rem;
  font-weight: 500;
  transition:
    transform 0.12s ease,
    box-shadow 0.12s ease,
    opacity 0.12s ease,
    background 0.12s ease;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 0.25rem;
}

button:hover {
  transform: translateY(-1px);
  box-shadow: 0 10px 20px rgba(16, 185, 129, 0.35);
}

button:disabled {
  opacity: 0.5;
  cursor: default;
  box-shadow: none;
  transform: none;
}

/* Boutons secondaires dans les cartes (Modifier / Supprimer) */
.item .row button:nth-child(2) {
  background: linear-gradient(135deg, #3b82f6, #2563eb);
}

.item .row button:nth-child(3),
.comments button:last-child {
  /* Supprimer / Ajouter un commentaire */
  background: linear-gradient(135deg, #ef4444, #b91c1c);
}

/* Carte d’un post */
.item {
  background: #020617;
  border-radius: 14px;
  padding: 1.25rem 1.25rem 1rem;
  margin-top: 0.75rem;
  box-shadow: 0 14px 30px rgba(0, 0, 0, 0.45);
  border: 1px solid rgba(55, 65, 81, 0.6);
}

.item h2 {
  margin: 0 0 0.15rem;
}

.item p {
  margin: 0.25rem 0;
  color: #e5e7eb;
  font-size: 0.95rem;
}

.item p:nth-of-type(3) {
  font-size: 0.85rem;
  color: #9ca3af;
}

/* Ligne de boutons d’un post (like / modifier / supprimer) */
.row {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  margin-top: 0.75rem;
}

/* Bloc édition d’un post */
.editBox {
  padding: 1rem;
  border-radius: 12px;
  border: 1px solid #10b981;
  background: radial-gradient(circle at top left, #064e3b, #020617);
  display: flex;
  flex-direction: column;
  gap: 0.6rem;
}

.editBox h3 {
  margin: 0 0 0.5rem;
}

/* Commentaires */
.comments {
  margin-top: 1rem;
  border-top: 1px solid #111827;
  padding-top: 0.75rem;
}

.comments h4 {
  margin: 0 0 0.4rem;
  font-size: 0.9rem;
  color: #d1d5db;
}

.comments ul {
  list-style: none;
  padding-left: 0;
  margin: 0 0 0.4rem;
}

.comments li {
  margin-bottom: 0.3rem;
  font-size: 0.85rem;
  color: #e5e7eb;
  display: flex;
  flex-direction: column;
  gap: 0.4rem;
}

.comments li strong {
  color: #a5b4fc;
}

/* Premier commentaire (aperçu) */
.first-comment {
  font-size: 0.88rem;
  margin-bottom: 0.4rem;
  color: #e5e7eb;
}

/* Formulaire d’ajout de commentaire */
.comment-form {
  display: flex;
  gap: 0.5rem;
  margin-top: 0.4rem;
}

/* Bouton "Afficher tous les commentaires" - discret */
.toggle-btn {
  background: transparent !important;
  color: #94a3b8;
  border: 1px solid #334155;
  font-size: 0.8rem;
  padding: 0.25rem 0.7rem;
  border-radius: 999px;
  opacity: 0.9;
  margin: 0.5rem 0 0.2rem;
  box-shadow: none !important;
}

.toggle-btn:hover {
  background: #020617 !important;
  color: #e5e7eb;
}

/* Responsive */
@media (max-width: 720px) {
  .container {
    padding: 1.25rem 1rem;
  }

  .form,
  .item {
    padding: 1rem;
  }

  .comment-form {
    flex-direction: column;
  }

  .actions,
  .search-bar {
    flex-direction: column;
    align-items: flex-start;
  }
}
</style>
