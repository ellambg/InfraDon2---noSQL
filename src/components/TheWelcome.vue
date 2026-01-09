<script setup lang="ts">
import { onMounted, ref, watch } from 'vue'
import PouchDB from 'pouchdb'
import PouchFind from 'pouchdb-find'

PouchDB.plugin(PouchFind)

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

const remoteDb = ref<PouchDB.Database | null>(null)
const localDb = ref<PouchDB.Database | null>(null)
const syncHandler = ref<PouchDB.Replication.Sync<{}> | null>(null)

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

const lastCommentByPost = ref<Record<string, CommentDoc | null>>({})
const otherCommentsByPost = ref<Record<string, CommentDoc[]>>({})

const fetchComments = async (postId: string) => {
  if (!localDb.value) return
  try {
    const res = await localDb.value.find({
      selector: { type: 'comment', postId },
      limit: 500,
    })

    const comments = (res.docs as CommentDoc[]).sort((a, b) =>
      b.created_at.localeCompare(a.created_at),
    )

    lastCommentByPost.value[postId] = comments[0] || null

    otherCommentsByPost.value[postId] = comments.slice(1)
  } catch (e) {
    console.error('Erreur fetchComments:', e)
  }
}

const attachmentUrlByPost = ref<Record<string, string | null>>({})

const PAGE_SIZE = 10
const postsBookmark = ref<string | null>(null)
const hasMorePosts = ref(true)

const nowIso = () => new Date().toISOString()
const uid = () => `${Date.now()}-${Math.random().toString(16).slice(2)}`

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

const getRemoteUrl = () => {
  const v = (import.meta as any).env?.VITE_COUCH_URL
  return v || 'http://admin:Infradon2_25!@localhost:5984/infradon2_db1'
}

const initDatabases = async () => {
  const remote = new PouchDB(getRemoteUrl())
  const local = new PouchDB('infradon2_local')
  remoteDb.value = remote
  localDb.value = local
  await createIndexes()
  await startLiveSync()
  await fetchPosts()
}

const createIndexes = async () => {
  if (!localDb.value) return
  await localDb.value.createIndex({ index: { fields: ['type', 'post_name'] } })
  await localDb.value.createIndex({ index: { fields: ['type', 'likes'] } })
  await localDb.value.createIndex({ index: { fields: ['type', 'created_at'] } })
  await localDb.value.createIndex({ index: { fields: ['type', 'postId', 'created_at'] } })
}

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
      await fetchPosts()
    })
    .on('error', (err) => console.error('Sync error:', err))
}

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

const revokeUrl = (postId: string) => {
  const url = attachmentUrlByPost.value[postId]
  if (url) URL.revokeObjectURL(url)
  attachmentUrlByPost.value[postId] = null
}

const loadAttachmentPreview = async (postId: string) => {
  if (!localDb.value) return
  try {
    const blob = (await localDb.value.getAttachment(postId, 'media')) as any
    if (!blob) {
      revokeUrl(postId)
      return
    }
    revokeUrl(postId)
    attachmentUrlByPost.value[postId] = URL.createObjectURL(blob as Blob)
  } catch {
    revokeUrl(postId)
  }
}

const addMediaToPost = async (post: PostDoc, file: File) => {
  if (!localDb.value || !post._id) return
  try {
    const latest = (await localDb.value.get(post._id)) as PostDoc
    await localDb.value.putAttachment(
      latest._id!,
      'media',
      latest._rev!,
      file,
      file.type || 'application/octet-stream',
    )
    await loadAttachmentPreview(post._id)
  } catch (e) {
    console.error('Erreur addMediaToPost:', e)
  }
}

const deleteMediaFromPost = async (post: PostDoc) => {
  if (!localDb.value || !post._id) return
  try {
    const latest: any = await localDb.value.get(post._id)
    await localDb.value.removeAttachment(post._id, 'media', latest._rev)
    revokeUrl(post._id)
  } catch (e) {
    console.error('Erreur deleteMediaFromPost:', e)
  }
}

const buildPostsQuery = (bookmark?: string | null) => {
  const term = searchTerm.value.trim()
  const selector: any = { type: 'post' as DocType }
  if (term) selector.post_name = { $regex: term }

  const query: any = { selector, limit: PAGE_SIZE }
  if (bookmark) query.bookmark = bookmark

  if (sortByLikes.value) {
    selector.likes = { $gte: 0 }
    query.sort = [{ type: 'asc' }, { likes: 'desc' }]
  } else {
    selector.created_at = { $gte: '' }
    query.sort = [{ type: 'asc' }, { created_at: 'desc' }]
  }
  return query
}

const fetchPosts = async () => {
  if (!localDb.value) return
  postsBookmark.value = null
  hasMorePosts.value = true

  try {
    const res: any = await localDb.value.find(buildPostsQuery(null))
    Object.keys(attachmentUrlByPost.value).forEach((id) => revokeUrl(id))
    postsData.value = res.docs as PostDoc[]
    postsBookmark.value = res.bookmark || null
    hasMorePosts.value = (postsData.value?.length || 0) === PAGE_SIZE

    await Promise.all(postsData.value.filter((p) => p._id).map((p) => fetchComments(p._id!)))
    await Promise.all(
      postsData.value.filter((p) => p._id).map((p) => loadAttachmentPreview(p._id!)),
    )
  } catch (e) {
    console.error('Erreur fetchPosts:', e)
  }
}

const loadMorePosts = async () => {
  if (!localDb.value || !hasMorePosts.value) return
  try {
    const res: any = await localDb.value.find(buildPostsQuery(postsBookmark.value))
    const nextDocs = res.docs as PostDoc[]
    postsData.value = [...postsData.value, ...nextDocs]
    postsBookmark.value = res.bookmark || null
    hasMorePosts.value = (nextDocs?.length || 0) === PAGE_SIZE

    await Promise.all(nextDocs.filter((p) => p._id).map((p) => fetchComments(p._id!)))
    await Promise.all(nextDocs.filter((p) => p._id).map((p) => loadAttachmentPreview(p._id!)))
  } catch (e) {
    console.error('Erreur loadMorePosts:', e)
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
    postsBookmark.value = null
    hasMorePosts.value = false

    await Promise.all(postsData.value.filter((p) => p._id).map((p) => fetchComments(p._id!)))
    await Promise.all(
      postsData.value.filter((p) => p._id).map((p) => loadAttachmentPreview(p._id!)),
    )
  } catch (e) {
    console.error('Erreur fetchTop10Liked:', e)
  }
}

const addPost = async () => {
  if (!localDb.value) return

  const post_name = newPost.value.post_name.trim()
  const post_content = newPost.value.post_content.trim()
  const attributes = (newPost.value.attributes || []).map((a) => String(a).trim()).filter(Boolean)

  if (!post_name || !post_content) return

  const doc: PostDoc = {
    _id: `post:${uid()}`,
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
    await putWithRetry<PostDoc>(doc, (latest) => ({
      ...latest,
      post_name: doc.post_name,
      post_content: doc.post_content,
      attributes: doc.attributes,
    }))
    editingPost.value = null
    await fetchPosts()
  } catch (e) {
    console.error('Erreur updatePost:', e)
  }
}

const deletePost = async (id?: string, rev?: string) => {
  if (!localDb.value || !id || !rev) return

  try {
    await localDb.value.remove(id, rev)

    const commRes = await localDb.value.find({
      selector: { type: 'comment', postId: id },
      limit: 500,
    })

    if (commRes.docs.length) {
      const toDelete = (commRes.docs as CommentDoc[]).map((c) => ({ ...c, _deleted: true }))
      await localDb.value.bulkDocs(toDelete as any)
    }

    revokeUrl(id)
    delete lastCommentByPost.value[id]
    delete otherCommentsByPost.value[id]
    delete commentsVisibility.value[id]

    await fetchPosts()
  } catch (e) {
    console.error('Erreur deletePost:', e)
  }
}

const likePost = async (post: PostDoc) => {
  if (!localDb.value || !post._id) return
  try {
    const latest = (await localDb.value.get(post._id)) as PostDoc

    await putWithRetry<PostDoc>(latest, (l) => ({ ...l, likes: (l.likes || 0) + 1 }))

    await fetchPosts()
  } catch (e) {
    console.error('Erreur likePost:', e)
  }
}

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
    _id: `comment:${uid()}`,
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
    await fetchComments(postId)
  } catch (e) {
    console.error('Erreur addComment:', e)
  }
}

const deleteComment = async (commentId: string, postId: string) => {
  if (!localDb.value) return
  try {
    const doc = (await localDb.value.get(commentId)) as CommentDoc
    await localDb.value.remove(doc._id!, doc._rev!)
    await fetchComments(postId)
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
    await putWithRetry<CommentDoc>(doc, (latest) => ({ ...latest, text: doc.text }))

    const postId = editingComment.value.postId
    editingComment.value = null
    await fetchComments(postId)
  } catch (e) {
    console.error('Erreur updateComment:', e)
  }
}

const toggleOtherComments = async (postId: string) => {
  commentsVisibility.value[postId] = !commentsVisibility.value[postId]
  // Optional: refresh comments when showing
  if (commentsVisibility.value[postId]) {
    await fetchComments(postId)
  }
}

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
    const id = `post:${uid()}`
    createdPostIds.push(id)
    docs.push({
      _id: id,
      ...createFakePost(i),
      created_at: nowIso(),
    } as PostDoc)
  }

  for (let i = 0; i < createdPostIds.length; i++) {
    const postId = createdPostIds[i]
    const howMany = 1 + Math.floor(Math.random() * 3)
    for (let j = 0; j < howMany; j++) {
      docs.push({
        _id: `comment:${uid()}`,
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
  <div class="app">
    <div class="wrap">
      <header class="top">
        <div class="brand">
          <div class="dot" />
          <div>
            <h1>InfraDon2</h1>
            <p>CouchDB / PouchDB + Vue 3</p>
          </div>
        </div>

        <div class="toolbar">
          <button class="btn" @click="fetchPosts">Rafraîchir</button>
          <button class="btn" @click="fullSyncOnce" :disabled="!isOnline">Sync manuel</button>

          <label class="toggle">
            <input type="checkbox" v-model="isOnline" />
            <span class="pill" :class="{ on: isOnline }">{{
              isOnline ? 'Online' : 'Offline'
            }}</span>
          </label>

          <button class="btn" @click="addManyFakePosts(10)">Factory</button>
          <button class="btn" @click="fetchTop10Liked">Top 10 likes</button>
        </div>
      </header>

      <section class="panel">
        <div class="search">
          <input v-model="searchTerm" type="text" placeholder="Rechercher par nom" />
          <label class="check">
            <input type="checkbox" v-model="sortByLikes" />
            <span>Trier par likes</span>
          </label>
        </div>

        <div class="create">
          <div class="head">
            <h2>Ajouter un post</h2>
          </div>

          <div class="grid">
            <div class="field">
              <span>Nom</span>
              <input v-model="newPost.post_name" placeholder="Nom du post" type="text" />
            </div>
            <div class="field">
              <span>Contenu</span>
              <input v-model="newPost.post_content" placeholder="Contenu du post" type="text" />
            </div>
            <div class="field full">
              <span>Attributs</span>
              <input
                :value="(newPost.attributes || []).join(', ')"
                placeholder="ex: demo, pair"
                type="text"
                @input="
                  newPost.attributes = ($event.target as HTMLInputElement).value
                    .split(',')
                    .map((s) => s.trim())
                    .filter(Boolean)
                "
              />
            </div>

            <button class="btn primary" @click="addPost">Ajouter</button>
          </div>
        </div>
      </section>

      <section v-if="postsData.length === 0" class="empty">
        <p>Aucune donnée trouvée.</p>
      </section>

      <section class="list">
        <article v-for="post in postsData" :key="post._id" class="card">
          <template v-if="!editingPost || editingPost._id !== post._id">
            <div class="cardTop">
              <div class="title">
                <h3>{{ post.post_name }}</h3>
                <div class="badges">
                  <span v-for="(a, idx) in post.attributes || []" :key="idx" class="badge">{{
                    a
                  }}</span>
                </div>
              </div>

              <div class="right">
                <div class="likes">
                  <span class="icon">❤️</span>
                  <span class="value">{{ post.likes || 0 }}</span>
                </div>

                <div class="actions">
                  <button class="btn" @click="startEdit(post)">Modifier</button>
                  <button class="btn danger" @click="deletePost(post._id, post._rev)">
                    Supprimer
                  </button>
                </div>
              </div>
            </div>

            <p class="content">{{ post.post_content }}</p>

            <div class="media" v-if="post._id">
              <div class="preview" v-if="attachmentUrlByPost[post._id]">
                <img :src="attachmentUrlByPost[post._id]!" alt="media" />
                <button class="btn danger sm" @click="deleteMediaFromPost(post)">
                  Supprimer le média
                </button>
              </div>

              <div class="upload">
                <span>Ajouter un média</span>
                <input
                  class="file"
                  type="file"
                  accept="image/*"
                  @change="
                    (e) => {
                      const f = (e.target as HTMLInputElement).files?.[0]
                      if (f) addMediaToPost(post, f)
                    }
                  "
                />
              </div>
            </div>

            <div class="comments">
              <div class="commentsTop">
                <h4>Commentaires</h4>
                <button
                  class="btn sm"
                  v-if="post._id && commentingPostId !== post._id"
                  @click="startComment(post._id!)"
                >
                  Ajouter
                </button>
              </div>

              <div v-if="post._id && lastCommentByPost[post._id]" class="last">
                <div class="line">
                  <span class="k">Dernier commentaire :</span>
                  <span class="v"
                    ><strong>{{ lastCommentByPost[post._id]!.author }}</strong> —
                    {{ lastCommentByPost[post._id]!.text }}</span
                  >
                </div>
                <div class="actions">
                  <button
                    class="btn sm"
                    @click="startEditComment(post._id!, lastCommentByPost[post._id]!)"
                  >
                    Modifier
                  </button>
                  <button
                    class="btn danger sm"
                    @click="deleteComment(lastCommentByPost[post._id]!._id!, post._id!)"
                  >
                    Supprimer
                  </button>
                </div>
              </div>

              <p v-else class="muted">Aucun commentaire.</p>

              <div v-if="post._id && (otherCommentsByPost[post._id]?.length ?? 0) > 0" class="more">
                <button class="btn ghost sm" @click="toggleOtherComments(post._id)">
                  {{ commentsVisibility[post._id] ? 'Masquer les autres' : 'Afficher les autres' }}
                </button>

                <ul v-if="commentsVisibility[post._id]" class="commentList">
                  <li v-for="c in otherCommentsByPost[post._id] || []" :key="c._id" class="comment">
                    <template v-if="editingComment && editingComment.commentId === c._id">
                      <input v-model="editingComment.text" type="text" />
                      <div class="actions">
                        <button class="btn primary sm" @click="updateComment">Enregistrer</button>
                        <button class="btn ghost sm" @click="cancelEditComment">Annuler</button>
                      </div>
                    </template>

                    <template v-else>
                      <div class="line">
                        <span class="v"
                          ><strong>{{ c.author }}</strong> — {{ c.text }}</span
                        >
                      </div>
                      <div class="actions">
                        <button class="btn sm" @click="startEditComment(post._id!, c)">
                          Modifier
                        </button>
                        <button class="btn danger sm" @click="deleteComment(c._id!, post._id!)">
                          Supprimer
                        </button>
                      </div>
                    </template>
                  </li>
                </ul>
              </div>

              <div v-if="commentingPostId === post._id" class="commentForm">
                <input v-model="newCommentText" type="text" placeholder="Votre commentaire" />
                <div class="actions">
                  <button class="btn primary sm" @click="addComment(post._id!)">Envoyer</button>
                  <button class="btn ghost sm" @click="commentingPostId = null">Annuler</button>
                </div>
              </div>
            </div>
          </template>

          <template v-else>
            <div class="edit">
              <div class="head">
                <h3>Modifier le post</h3>
              </div>

              <div class="grid">
                <div class="field">
                  <span>Nom</span>
                  <input v-model="editingPost.post_name" type="text" />
                </div>
                <div class="field">
                  <span>Contenu</span>
                  <input v-model="editingPost.post_content" type="text" />
                </div>
                <div class="field full">
                  <span>Attributs</span>
                  <input
                    :value="(editingPost.attributes || []).join(', ')"
                    type="text"
                    @input="
                      editingPost &&
                      (editingPost.attributes = ($event.target as HTMLInputElement).value
                        .split(',')
                        .map((s) => s.trim())
                        .filter(Boolean))
                    "
                  />
                </div>
              </div>

              <div class="actions">
                <button class="btn primary" @click="updatePost">Enregistrer</button>
                <button class="btn ghost" @click="cancelEdit">Annuler</button>
              </div>
            </div>
          </template>
        </article>

        <div class="pager" v-if="postsData.length > 0">
          <button class="btn" @click="loadMorePosts" :disabled="!hasMorePosts">
            Afficher les 10 suivants
          </button>
        </div>
      </section>
    </div>
  </div>
</template>

<style scoped>
/* Thème sombre de style « Instagram » */
.app {
  min-height: 100vh;
  background: #121212; /* fond très sombre */
  color: #f5f5f5; /* texte clair */
}

/* Barre d’en-tête */
.top {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: 16px;
  padding: 16px 18px;
  margin-bottom: 20px;
  border-radius: 16px;
  background: #181818;
  border: 1px solid #262626;
}

/* Panneau (recherche + ajout de post) */
.panel {
  background: #181818;
  border: 1px solid #262626;
  border-radius: 20px;
  padding: 20px;
  margin-bottom: 28px;
}

/* Cartes de posts */
.card {
  background: #181818;
  border: 1px solid #262626;
  border-radius: 16px;
  padding: 16px;
}

/* Boutons génériques */
.btn {
  border-radius: 999px;
  padding: 8px 14px;
  font-size: 13px;
  background: #262626;
  color: #f5f5f5;
  border: 1px solid #3f3f46;
  cursor: pointer;
}
.btn:hover {
  background: #333333;
}

/* Bouton principal (rose/violet à la manière d’Instagram) */
.btn.primary {
  background: #e1306c;
  border-color: #e1306c;
  color: #f5f5f5;
}
.btn.primary:hover {
  background: #c72e63;
}

/* Bouton danger */
.btn.danger {
  background: #dc3545;
  border-color: #dc3545;
  color: #f5f5f5;
}
.btn.danger:hover {
  background: #b02a37;
}

/* Boutons fantômes */
.btn.ghost {
  border: none;
  background: transparent;
  color: #f5f5f5;
}
.btn.ghost:hover {
  background: #262626;
}

/* Boutons de petite taille */
.btn.sm {
  padding: 6px 10px;
  font-size: 12px;
}

/* Champs de saisie */
input {
  padding: 10px 12px;
  border-radius: 10px;
  border: 1px solid #262626;
  background: #181818;
  color: #f5f5f5;
  font-size: 13px;
}
input::placeholder {
  color: #6b7280;
}
input:focus {
  outline: none;
  border-color: #e1306c;
}

/* Section liste */
.list {
  display: flex;
  flex-direction: column;
  gap: 14px;
  border-top: 1px solid #262626;
  padding-top: 24px;
}

/* Commentaires et « dernier commentaire » */
.last,
.comment {
  margin-top: 10px;
  padding: 10px;
  background: #202020;
  border: 1px solid #333333;
  border-radius: 12px;
}

/* État vide */
.empty {
  padding: 20px;
  border-radius: 16px;
  border: 1px dashed #333333;
  color: #6b7280;
  text-align: center;
}

/* Zone des likes avec icône */
.likes {
  display: flex;
  align-items: center;
  gap: 6px;
}
.likes .icon {
  color: #e1306c;
  font-size: 16px;
}
.likes .value {
  color: #f5f5f5;
  font-weight: bold;
}
</style>
