<script setup lang="ts">
import { onMounted, ref, watch } from 'vue'
import PouchDB from 'pouchdb'
import PouchFind from 'pouchdb-find'

PouchDB.plugin(PouchFind)

/** ========= Types ========= */
type DocType = 'post' | 'comment'

interface PostDoc {
  _id?: string
  _rev?: string
  type: 'post'
  post_name: string
  post_content: string
  attributes: string[]
  likes: number
  created_at: string
}

interface CommentDoc {
  _id?: string
  _rev?: string
  type: 'comment'
  postId: string
  author: string
  text: string
  created_at: string
}

/** ========= DB ========= */
const remoteDb = ref<PouchDB.Database | null>(null)
const localDb = ref<PouchDB.Database | null>(null)
const syncHandler = ref<PouchDB.Replication.Sync<{}> | null>(null)

/** ========= UI state ========= */
const postsData = ref<PostDoc[]>([])

const newPost = ref({
  post_name: '',
  post_content: '',
  attributes: [] as string[],
})

const editingPost = ref<PostDoc | null>(null)

const searchTerm = ref('')
const sortByLikes = ref(false)
const isOnline = ref(true)

const commentingPostId = ref<string | null>(null)
const newCommentText = ref('')

const editingComment = ref<{
  postId: string
  commentId: string
  text: string
} | null>(null)

const commentsVisibility = ref<Record<string, boolean>>({})

/** ========= Comment cache (minimal data) ========= */
const firstCommentByPost = ref<Record<string, CommentDoc | null>>({})
const otherCommentsByPost = ref<Record<string, CommentDoc[]>>({})

/** ========= Helpers ========= */
const nowIso = () => new Date().toISOString()

// petit helper anti-conflit : re-get et retry (utile pour likes, edits)
const putWithRetry = async <T extends { _id?: string; _rev?: string }>(
  draft: T,
  merge: (latest: T) => T,
  maxRetry = 2,
) => {
  if (!localDb.value || !draft._id) throw new Error('DB or _id missing')
  try {
    return await localDb.value.put(draft as any)
  } catch (e: any) {
    if (e?.status === 409 && maxRetry > 0) {
      const latest = (await localDb.value.get(draft._id)) as T
      const merged = merge(latest)
      merged._id = latest._id
      merged._rev = latest._rev
      return await putWithRetry(merged, merge, maxRetry - 1)
    }
    throw e
  }
}

/** ========= Init / Indexes ========= */
const initDatabases = async () => {
  const remote = new PouchDB('http://admin:Infradon2_25!@localhost:5984/infradon2_db1')
  const local = new PouchDB('infradon2_local')

  remoteDb.value = remote
  localDb.value = local

  await createIndexes()
  await startLiveSync()
  await fetchPosts()
}

const createIndexes = async () => {
  if (!localDb.value) return

  // Posts: search
  await localDb.value.createIndex({
    index: { fields: ['type', 'post_name'] },
  })

  // Posts: sort likes
  await localDb.value.createIndex({
    index: { fields: ['type', 'likes'] },
  })

  // Comments: by post + date
  await localDb.value.createIndex({
    index: { fields: ['type', 'postId', 'created_at'] },
  })
}

/** ========= Sync (plus robuste que replicate "one shot") ========= */
const stopLiveSync = () => {
  if (syncHandler.value) {
    syncHandler.value.cancel()
    syncHandler.value = null
  }
}

const startLiveSync = async () => {
  if (!localDb.value || !remoteDb.value) return
  stopLiveSync()

  syncHandler.value = localDb.value
    .sync(remoteDb.value, { live: true, retry: true })
    .on('change', async () => {
      // refresh simple et fiable
      await fetchPosts()
    })
    .on('error', (err) => console.error('Sync error:', err))
}

// bouton sync manuel si tu veux le garder
const fullSyncOnce = async () => {
  if (!localDb.value || !remoteDb.value) return
  try {
    await PouchDB.replicate(localDb.value, remoteDb.value)
    await PouchDB.replicate(remoteDb.value, localDb.value)
    await fetchPosts()
  } catch (e) {
    console.error('Erreur fullSyncOnce:', e)
  }
}

/** ========= Queries (NO allDocs) ========= */
const fetchPosts = async () => {
  if (!localDb.value) return

  const term = searchTerm.value.trim()
  const selector: any = { type: 'post' as DocType }

  if (term) selector.post_name = { $regex: term }

  const query: any = {
    selector,
    limit: 200, // IMPORTANT: pas 25 par défaut
  }

  // sort by likes
  if (sortByLikes.value) {
    selector.likes = { $gte: 0 }
    query.sort = [{ type: 'asc' }, { likes: 'desc' }]
  }

  try {
    const res = await localDb.value.find(query)
    postsData.value = res.docs as PostDoc[]
    await fetchFirstCommentsForPosts(postsData.value)
  } catch (e) {
    console.error('Erreur fetchPosts:', e)
  }
}

const fetchTop10Liked = async () => {
  if (!localDb.value) return
  try {
    const res = await localDb.value.find({
      selector: { type: 'post', likes: { $gte: 0 } },
      sort: [{ type: 'asc' }, { likes: 'desc' }],
      limit: 10,
    })
    postsData.value = res.docs as PostDoc[]
    await fetchFirstCommentsForPosts(postsData.value)
  } catch (e) {
    console.error('Erreur fetchTop10Liked:', e)
  }
}

/** ========= Posts CRUD ========= */
const addPost = async () => {
  if (!localDb.value) return

  const post_name = newPost.value.post_name.trim()
  const post_content = newPost.value.post_content.trim()
  const attributes = (newPost.value.attributes || []).map((a) => String(a).trim()).filter(Boolean)

  if (!post_name || !post_content) return

  const doc: PostDoc = {
    _id: `post:${nowIso()}`,
    type: 'post',
    post_name,
    post_content,
    attributes,
    likes: 0,
    created_at: nowIso(),
  }

  try {
    await localDb.value.put(doc)
    newPost.value.post_name = ''
    newPost.value.post_content = ''
    newPost.value.attributes = []
    await fetchPosts()
  } catch (e) {
    console.error('Erreur addPost:', e)
  }
}

const startEdit = (post: PostDoc) => {
  editingPost.value = JSON.parse(JSON.stringify(post))
}
const cancelEdit = () => (editingPost.value = null)

const updatePost = async () => {
  if (!localDb.value) return
  const doc = editingPost.value
  if (!doc?._id || !doc._rev) return

  doc.post_name = doc.post_name.trim()
  doc.post_content = doc.post_content.trim()
  doc.attributes = (doc.attributes || []).map((a) => String(a).trim()).filter(Boolean)

  try {
    await putWithRetry<PostDoc>(
      doc,
      (latest) => ({
        ...latest,
        post_name: doc.post_name,
        post_content: doc.post_content,
        attributes: doc.attributes,
      }),
    )
    editingPost.value = null
    await fetchPosts()
  } catch (e) {
    console.error('Erreur updatePost:', e)
  }
}

const deletePost = async (id?: string, rev?: string) => {
  if (!localDb.value || !id || !rev) return

  try {
    // delete post
    await localDb.value.remove(id, rev)

    // delete all linked comments (collection séparée)
    const commRes = await localDb.value.find({
      selector: { type: 'comment', postId: id },
      limit: 500,
    })

    if (commRes.docs.length) {
      const toDelete = (commRes.docs as CommentDoc[]).map((c) => ({ ...c, _deleted: true }))
      await localDb.value.bulkDocs(toDelete as any)
    }

    await fetchPosts()
  } catch (e) {
    console.error('Erreur deletePost:', e)
  }
}

const likePost = async (post: PostDoc) => {
  if (!localDb.value || !post._id) return
  try {
    const latest = (await localDb.value.get(post._id)) as PostDoc
    latest.likes = (latest.likes || 0) + 1
    await putWithRetry<PostDoc>(
      latest,
      (l) => ({ ...l, likes: (l.likes || 0) + 1 }),
    )
    await fetchPosts()
  } catch (e) {
    console.error('Erreur likePost:', e)
  }
}

/** ========= Comments ========= */
const startComment = (postId: string) => {
  commentingPostId.value = postId
  newCommentText.value = ''
  editingComment.value = null
}

const addComment = async (postId: string) => {
  if (!localDb.value) return
  const txt = newCommentText.value.trim()
  if (!txt) return

  const doc: CommentDoc = {
    _id: `comment:${nowIso()}`,
    type: 'comment',
    postId,
    author: 'Anonyme',
    text: txt,
    created_at: nowIso(),
  }

  try {
    await localDb.value.put(doc)
    newCommentText.value = ''
    commentingPostId.value = null

    await fetchFirstComment(postId)
    if (commentsVisibility.value[postId]) await fetchOtherComments(postId)
  } catch (e) {
    console.error('Erreur addComment:', e)
  }
}

const deleteComment = async (commentId: string, postId: string) => {
  if (!localDb.value) return
  try {
    const doc = (await localDb.value.get(commentId)) as CommentDoc
    await localDb.value.remove(doc._id!, doc._rev!)
    await fetchFirstComment(postId)
    if (commentsVisibility.value[postId]) await fetchOtherComments(postId)
  } catch (e) {
    console.error('Erreur deleteComment:', e)
  }
}

const startEditComment = (postId: string, comment: CommentDoc) => {
  if (!comment._id) return
  editingComment.value = { postId, commentId: comment._id, text: comment.text }
  commentingPostId.value = null
}

const cancelEditComment = () => (editingComment.value = null)

const updateComment = async () => {
  if (!localDb.value || !editingComment.value) return
  try {
    const doc = (await localDb.value.get(editingComment.value.commentId)) as CommentDoc
    doc.text = editingComment.value.text.trim()
    await putWithRetry<CommentDoc>(
      doc,
      (latest) => ({ ...latest, text: doc.text }),
    )

    const postId = editingComment.value.postId
    editingComment.value = null
    await fetchFirstComment(postId)
    if (commentsVisibility.value[postId]) await fetchOtherComments(postId)
  } catch (e) {
    console.error('Erreur updateComment:', e)
  }
}

/** ========= Minimal comments loading ========= */
const fetchFirstComment = async (postId: string) => {
  if (!localDb.value) return
  try {
    const res = await localDb.value.find({
      selector: { type: 'comment', postId },
      sort: [{ type: 'asc' }, { postId: 'asc' }, { created_at: 'asc' }],
      limit: 1,
    })
    firstCommentByPost.value[postId] = (res.docs[0] as CommentDoc) || null
  } catch (e) {
    console.error('Erreur fetchFirstComment:', e)
  }
}

const fetchFirstCommentsForPosts = async (posts: PostDoc[]) => {
  await Promise.all(posts.filter(p => p._id).map(p => fetchFirstComment(p._id!)))
}

const fetchOtherComments = async (postId: string) => {
  if (!localDb.value) return
  try {
    const res = await localDb.value.find({
      selector: { type: 'comment', postId },
      sort: [{ type: 'asc' }, { postId: 'asc' }, { created_at: 'asc' }],
      limit: 500,
    })
    const all = res.docs as CommentDoc[]
    otherCommentsByPost.value[postId] = all.slice(1) // exclude first
  } catch (e) {
    console.error('Erreur fetchOtherComments:', e)
  }
}

const toggleOtherComments = async (postId: string) => {
  commentsVisibility.value[postId] = !commentsVisibility.value[postId]
  if (commentsVisibility.value[postId]) {
    await fetchOtherComments(postId)
  }
}

/** ========= Factory (DB vide -> fonctionne) ========= */
const createFakePost = (i: number): Omit<PostDoc, '_id' | '_rev'> => ({
  type: 'post',
  post_name: `Message ${i}`,
  post_content: `Contenu du message ${i}`,
  attributes: ['demo', i % 2 === 0 ? 'pair' : 'impair'],
  likes: Math.floor(Math.random() * 20),
  created_at: nowIso(),
})

const addManyFakePosts = async (count = 10) => {
  if (!localDb.value) return

  const docs: any[] = []
  const createdPostIds: string[] = []

  for (let i = 0; i < count; i++) {
    const id = `post:${nowIso()}-${i}`
    createdPostIds.push(id)
    docs.push({
      _id: id,
      ...createFakePost(i),
      created_at: nowIso(),
    } as PostDoc)
  }

  // ajoute aussi quelques commentaires séparés
  for (let i = 0; i < createdPostIds.length; i++) {
    const postId = createdPostIds[i]
    const howMany = 1 + Math.floor(Math.random() * 3) // 1 à 3 commentaires
    for (let j = 0; j < howMany; j++) {
      docs.push({
        _id: `comment:${nowIso()}-${i}-${j}`,
        type: 'comment',
        postId,
        author: 'Anonyme',
        text: `Commentaire ${j + 1} sur ${postId.split(':')[1]}`,
        created_at: nowIso(),
      } as CommentDoc)
    }
  }

  try {
    await localDb.value.bulkDocs(docs)
    await fetchPosts()
  } catch (e) {
    console.error('Erreur addManyFakePosts:', e)
  }
}

/** ========= Watchers / lifecycle ========= */
watch([searchTerm, sortByLikes], () => {
  fetchPosts()
})

watch(isOnline, async (v) => {
  if (v) await startLiveSync()
  else stopLiveSync()
})

onMounted(async () => {
  await initDatabases()
})
</script>

<template>
  <div class="container">
    <h1>InfraDon - CouchDB + Vue 3</h1>

    <div class="actions">
      <button @click="fetchPosts">Rafraîchir (find)</button>
      <button @click="fullSyncOnce" :disabled="!isOnline">Sync manuel</button>

      <label class="inline-label">
        <input type="checkbox" v-model="isOnline" />
        Mode online
      </label>

      <button @click="addManyFakePosts(10)">Factory (10 posts + comments)</button>
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

        <!-- COMMENTS -->
        <div class="comments">
          <h4>Commentaires</h4>

          <!-- Premier commentaire (1 seule fois) -->
          <div v-if="post._id && firstCommentByPost[post._id]" class="first-comment">
            <p>
              <strong>Premier commentaire :</strong><br />
              <strong>{{ firstCommentByPost[post._id]!.author }}</strong>
              — {{ firstCommentByPost[post._id]!.text }}
            </p>

            <div class="row">
              <button
                @click="startEditComment(post._id!, firstCommentByPost[post._id]!)"
              >
                Modifier
              </button>
              <button
                @click="deleteComment(firstCommentByPost[post._id]!._id!, post._id!)"
              >
                Supprimer
              </button>
            </div>
          </div>

          <p v-else>Aucun commentaire pour l’instant.</p>

          <!-- Afficher autres -->
          <div v-if="post._id && (otherCommentsByPost[post._id]?.length ?? 0) > 0">
            <button class="toggle-btn" @click="toggleOtherComments(post._id)">
              {{ commentsVisibility[post._id] ? 'Masquer les autres commentaires' : 'Afficher les autres commentaires' }}
            </button>

            <ul v-if="commentsVisibility[post._id]">
              <li v-for="c in (otherCommentsByPost[post._id] || [])" :key="c._id">
                <!-- edition -->
                <template v-if="editingComment && editingComment.commentId === c._id">
                  <input v-model="editingComment.text" type="text" />
                  <div class="row">
                    <button @click="updateComment">Enregistrer</button>
                    <button @click="cancelEditComment">Annuler</button>
                  </div>
                </template>

                <!-- normal -->
                <template v-else>
                  <span><strong>{{ c.author }}</strong> — {{ c.text }}</span>
                  <div class="row">
                    <button @click="startEditComment(post._id!, c)">Modifier</button>
                    <button @click="deleteComment(c._id!, post._id!)">Supprimer</button>
                  </div>
                </template>
              </li>
            </ul>
          </div>

          <!-- Formulaire d’ajout -->
          <div v-if="commentingPostId === post._id" class="comment-form">
            <input v-model="newCommentText" type="text" placeholder="Votre commentaire" />
            <button @click="addComment(post._id!)">Envoyer</button>
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
  flex-wrap: wrap;
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
  transition: border-color 0.15s ease, box-shadow 0.15s ease, background 0.15s ease;
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
  transition: transform 0.12s ease, box-shadow 0.12s ease, opacity 0.12s ease,
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

/* Boutons secondaires dans les cartes */
.item .row button:nth-child(2) {
  background: linear-gradient(135deg, #3b82f6, #2563eb);
}

.item .row button:nth-child(3),
.comments button:last-child {
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

.row {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  margin-top: 0.75rem;
}

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

.first-comment {
  font-size: 0.88rem;
  margin-bottom: 0.4rem;
  color: #e5e7eb;
}

.comment-form {
  display: flex;
  gap: 0.5rem;
  margin-top: 0.4rem;
}

/* Bouton discret "Afficher les autres commentaires" */
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
