<script setup lang="ts">
import { ref, onMounted } from "vue"
import PouchDB from "pouchdb"
import PouchFind from "pouchdb-find"

PouchDB.plugin(PouchFind)

/*
  offline-firstes donc les base distantes servent uniquement pour la réplication. */
const postsDB = ref<any>(null)
const commentsDB = ref<any>(null)
const postsRemote = ref<any>(null)
const commentsRemote = ref<any>(null)

const posts = ref<any[]>([])
const comments = ref<Record<string, any[]>>({})
const lastComments = ref<Record<string, any | null>>({})

const newTitle = ref("")
const newContent = ref("")
const searchText = ref("")
const newComment = ref<Record<string, string>>({})
const editComment = ref<Record<string, string>>({})
const editingCommentId = ref<string | null>(null)
const factoryCount = ref<number>(20)
const online = ref(true)


const page = ref(0)
const PAGE_SIZE = 10


const fileInput = ref<Record<string, File | null>>({})

const initDB = () => {
  postsDB.value = new PouchDB("posts-local")
  commentsDB.value = new PouchDB("comments-local")
  postsRemote.value = new PouchDB("http://admin:matteo@localhost:5984/posts")
  commentsRemote.value = new PouchDB("http://admin:matteo@localhost:5984/comments")
}

/*
  indexer pour éviter d'utiliser allDocs(include_docs) et de filtré en js */
const createIndexes = async () => {
  await postsDB.value.createIndex({ index: { fields: ["createdAt"] } })
  await postsDB.value.createIndex({ index: { fields: ["title"] } })
  await postsDB.value.createIndex({ index: { fields: ["likes"] } })
  await commentsDB.value.createIndex({ index: { fields: ["postId", "createdAt"] } })
}

const loadPosts = async () => {
  const result = await postsDB.value.find({
    selector: { createdAt: { $gte: null } },
    sort: [{ createdAt: "desc" }],
    limit: PAGE_SIZE,
    skip: page.value * PAGE_SIZE
  })
  posts.value = result.docs
}

const nextPage = async () => {
  page.value++
  await loadPosts()
}

const prevPage = async () => {
  if (page.value > 0) {
    page.value--
    await loadPosts()
  }
}

/* seulement le dernier commentaire */
const loadLastComment = async (postId: string) => {
  const result = await commentsDB.value.find({
    selector: { postId },
    sort: [{ createdAt: "desc" }],
    limit: 1
  })
  lastComments.value[postId] = result.docs[0] || null
}

const loadCommentsForPost = async (postId: string) => {
  const result = await commentsDB.value.find({
    selector: { postId },
    sort: [{ createdAt: "asc" }]
  })
  comments.value[postId] = result.docs
}

const addPost = async () => {
  if (!newTitle.value || !newContent.value) return
  await postsDB.value.put({
    _id: crypto.randomUUID(),
    title: newTitle.value,
    content: newContent.value,
    likes: 0,
    createdAt: new Date().toISOString()
  })
  newTitle.value = ""
  newContent.value = ""
  await loadPosts()
}

const updatePost = async (post: any) => {
  await postsDB.value.put({ ...post })
  await loadPosts()
}

const deletePost = async (post: any) => {
  await postsDB.value.remove(post)
  await loadPosts()
}

const likePost = async (post: any) => {
  await postsDB.value.put({ ...post, likes: post.likes + 1 })
  await loadPosts()
}

const attachFile = async (post: any) => {
  const file = fileInput.value[post._id]
  if (!file) return
  const doc = await postsDB.value.get(post._id)
  await postsDB.value.putAttachment(doc._id, "media", doc._rev, file, file.type)
}

const removeFile = async (post: any) => {
  const doc = await postsDB.value.get(post._id)
  await postsDB.value.removeAttachment(doc._id, "media", doc._rev)
}

const addComment = async (postId: string) => {
  const text = newComment.value[postId]
  if (!text) return
  await commentsDB.value.put({
    _id: crypto.randomUUID(),
    postId,
    text,
    createdAt: new Date().toISOString()
  })
  newComment.value[postId] = ""
  await loadCommentsForPost(postId)
  await loadLastComment(postId)
}

const startEditComment = (comment: any) => {
  editingCommentId.value = comment._id
  editComment.value[comment._id] = comment.text
}

const updateComment = async (comment: any) => {
  await commentsDB.value.put({ ...comment, text: editComment.value[comment._id] })
  editingCommentId.value = null
  await loadCommentsForPost(comment.postId)
}

const deleteComment = async (comment: any) => {
  await commentsDB.value.remove(comment)
  await loadCommentsForPost(comment.postId)
}

const searchPosts = async () => {
  if (!searchText.value) {
    await loadPosts()
    return
  }
  const result = await postsDB.value.find({
    selector: { title: searchText.value }
  })
  posts.value = result.docs
}

const sortByLikes = async () => {
  const result = await postsDB.value.find({
    selector: { likes: { $gte: 0 } },
    sort: [{ likes: "desc" }],
    limit: 10
  })
  posts.value = result.docs
}

const startSync = () => {
  postsDB.value.sync(postsRemote.value, { live: true, retry: true })
    .on("paused", () => online.value = false)
    .on("active", () => online.value = true)

  commentsDB.value.sync(commentsRemote.value, { live: true, retry: true })
}

onMounted(async () => {
  initDB()
  await createIndexes()
  await loadPosts()
  startSync()
})
</script>

<template>
  <h1>CouchDB + Vue + PouchDB</h1>

  <p :style="{ color: online ? 'green' : 'red' }">
    ● Statut : <strong>{{ online ? "Online" : "Offline" }}</strong>
  </p>

  <h2>Ajouter un post</h2>
  <input v-model="newTitle" placeholder="Titre" />
  <input v-model="newContent" placeholder="Contenu" />
  <button @click="addPost">Ajouter</button>

  <hr>

  <h2>Recherche</h2>
  <input v-model="searchText" placeholder="Titre exact" />
  <button @click="searchPosts">Rechercher</button>

  <hr>

  <h2>Tri</h2>
  <button @click="sortByLikes">Top 10 les plus likés</button>

  <hr>

  <h2>Pagination</h2>
  <button @click="prevPage">Précédent</button>
  <button @click="nextPage">Suivant</button>

  <hr>

  <h2>Factory</h2>
  <input v-model.number="factoryCount" type="number" />
  <button @click="generateFakePosts">Générer</button>

  <hr>

  <div v-for="post in posts" :key="post._id" class="post">
    <h3>{{ post.title }}</h3>
    <p>{{ post.content }}</p>

    <input v-model="post.content" />
    <button @click="updatePost(post)">Mettre à jour</button>
    <button @click="deletePost(post)">Supprimer</button>

    <p>Likes : {{ post.likes }}</p>
    <button @click="likePost(post)">Like</button>

    <div>
      <input type="file" @change="e => fileInput[post._id] = e.target.files[0]" />
      <button @click="attachFile(post)">Ajouter média</button>
      <button @click="removeFile(post)">Supprimer média</button>
    </div>

    <p v-if="lastComments[post._id]">
      Dernier commentaire : {{ lastComments[post._id].text }}
    </p>
    <button @click="loadLastComment(post._id)">Charger dernier commentaire</button>

    <div class="comments">
      <button @click="loadCommentsForPost(post._id)">Voir tous les commentaires</button>

      <ul>
        <li v-for="c in comments[post._id]" :key="c._id">
          <span v-if="editingCommentId !== c._id">{{ c.text }}</span>
          <input v-else v-model="editComment[c._id]" />
          <button v-if="editingCommentId !== c._id" @click="startEditComment(c)">Modifier</button>
          <button v-else @click="updateComment(c)">Valider</button>
          <button @click="deleteComment(c)">Supprimer</button>
        </li>
      </ul>

      <input v-model="newComment[post._id]" placeholder="Nouveau commentaire" />
      <button @click="addComment(post._id)">Ajouter</button>
    </div>
  </div>
</template>

<style scoped>
.post {
  border: 1px solid #ccc;
  padding: 15px;
  margin-bottom: 15px;
}
.comments {
  border-left: 3px solid #42b883;
  padding-left: 10px;
}
</style>

J’ai privilégier comme demandé des échanges efficaces en évitant donc de répliquer inutilement quoi que ce soit, 
l’utilisation de allDocs(include_docs) et les filtres etc en js. les données sont indexées et chargée que à la 
demande et synchronisée non-stop pour que ce soit cohérent même en mode hors ligne. les méthodes les plus utiles 
selon moi sont createIndex(), find(), limit(), skip() et sort().