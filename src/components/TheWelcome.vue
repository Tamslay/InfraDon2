<script setup lang="ts">
import { ref, onMounted } from "vue"
import PouchDB from "pouchdb"
import PouchFind from "pouchdb-find"

// Activation du plugin Find (index + query)
PouchDB.plugin(PouchFind)

// --- BASES DE DONNÉES ---
const localDB = ref<any>(null)
const remoteDB = ref<any>(null)

// --- DONNÉES ---
const posts = ref<any[]>([])
const searchText = ref("")
const newTitle = ref("")
const newContent = ref("")
const factoryCount = ref(20)

// --- COMMENTAIRES / LIKES ---
const newComment = ref("")
const editingContent = ref("")
const online = ref(true)

// -------------------------------------------------------
// 🔹 INITIALISATION DES BASES
// -------------------------------------------------------
const initDB = () => {
  localDB.value = new PouchDB("posts-local")
  remoteDB.value = new PouchDB("http://admin:matteo@localhost:5984/message")
  console.log("✅ Base locale + distante initialisées")
}

// -------------------------------------------------------
// 🔹 FETCH
// -------------------------------------------------------
const loadPosts = async () => {
  if (!localDB.value) return

  const result = await localDB.value.allDocs({ include_docs: true })
  posts.value = result.rows.map(r => r.doc)
}

// -------------------------------------------------------
// 🔹 AJOUT DE POST
// -------------------------------------------------------
const addPost = async () => {
  if (!newTitle.value || !newContent.value) return

  const doc = {
    _id: new Date().toISOString(),
    title: newTitle.value,
    content: newContent.value,
    likes: 0,
    comments: [],
    createdAt: new Date().toISOString()
  }

  await localDB.value.put(doc)
  newTitle.value = ""
  newContent.value = ""
  loadPosts()
}

// -------------------------------------------------------
// 🔹 SUPPRIMER
// -------------------------------------------------------
const deletePost = async (post: any) => {
  await localDB.value.remove(post)
  loadPosts()
}

// -------------------------------------------------------
// 🔹 UPDATE
// -------------------------------------------------------
const updatePost = async (post: any) => {
  const updated = { ...post }
  await localDB.value.put(updated)
  loadPosts()
}

// -------------------------------------------------------
// 🔹 LIKE 👍
// -------------------------------------------------------
const likePost = async (post: any) => {
  post.likes++
  await localDB.value.put(post)
  loadPosts()
}

// -------------------------------------------------------
// 🔹 AJOUT COMMENTAIRE
// -------------------------------------------------------
const addComment = async (post: any) => {
  if (!newComment.value) return

  post.comments.push({
    text: newComment.value,
    date: new Date().toISOString()
  })

  await localDB.value.put(post)
  newComment.value = ""
  loadPosts()
}

// -------------------------------------------------------
// 🔹 RECHERCHE VIA INDEX + FIND
// -------------------------------------------------------
const createIndex = async () => {
  await localDB.value.createIndex({
    index: { fields: ["title"] }
  })
  console.log("📌 Index créé sur 'title'")
}

const searchPosts = async () => {
  if (!searchText.value) return loadPosts()

  const result = await localDB.value.find({
    selector: { title: { $regex: searchText.value } }
  })

  posts.value = result.docs
}

// -------------------------------------------------------
// 🔹 FACTORY — Création massive d’objets
// -------------------------------------------------------
const generateFakePosts = async () => {
  const batch = []

  for (let i = 0; i < factoryCount.value; i++) {
    batch.push({
      _id: new Date().getTime().toString() + "-" + i,
      title: "Fake Post " + i,
      content: "Lorem ipsum " + Math.random(),
      likes: Math.floor(Math.random() * 50),
      comments: [],
      createdAt: new Date().toISOString()
    })
  }

  await localDB.value.bulkDocs(batch)
  console.log("⚙️  Factory exécutée :", factoryCount.value, "posts")
  loadPosts()
}

// -------------------------------------------------------
// 🔹 SYNCHRO TEMPS RÉEL
// -------------------------------------------------------
const startSync = () => {
  localDB.value.sync(remoteDB.value, {
    live: true,
    retry: true
  })
    .on("change", info => {
      console.log("🔄 Sync change :", info)
      loadPosts()
    })
    .on("paused", () => online.value = false)
    .on("active", () => online.value = true)
    .on("error", err => console.error("❌ Sync error :", err))

  console.log("🔁 Synchronisation temps réel activée")
}

// -------------------------------------------------------
// 🔹 TRACK DES CHANGEMENTS (live reload local)
// -------------------------------------------------------
const listenLocalChanges = () => {
  localDB.value.changes({
    since: "now",
    live: true,
    include_docs: true
  }).on("change", loadPosts)
}

// -------------------------------------------------------
// 🔹 MONTAGE
// -------------------------------------------------------
onMounted(() => {
  initDB()
  loadPosts()
  createIndex()
  listenLocalChanges()
  startSync()
})
</script>

<template>
  <h1>💬 CouchDB + Vue.js + PouchDB</h1>

  <p :style="{color: online ? 'green' : 'red'}">
    ● Statut : <strong>{{ online ? "Online" : "Offline" }}</strong>
  </p>

  <!-- AJOUT -->
  <h2>➕ Ajouter un post</h2>
  <input v-model="newTitle" placeholder="Titre du post" />
  <input v-model="newContent" placeholder="Contenu du post" />
  <button @click="addPost">Ajouter</button>

  <hr>

  <!-- RECHERCHE -->
  <h2>🔍 Rechercher un post</h2>
  <input v-model="searchText" placeholder="Recherche titre" />
  <button @click="searchPosts">Rechercher</button>

  <hr>

  <!-- SYNCHRO -->
  <button @click="startSync">🔄 Synchroniser vers CouchDB</button>

  <hr>

  <!-- FACTORY -->
  <h2>⚙️ Générer des posts (Factory)</h2>
  <input v-model="factoryCount" type="number" />
  <button @click="generateFakePosts">Générer</button>

  <hr>

  <!-- LISTE DES POSTS -->
  <h2>📚 Liste des posts</h2>

  <div v-for="post in posts" :key="post._id" class="post">
    <h3>{{ post.title }}</h3>
    <p>{{ post.content }}</p>

    <input v-model="post.content" />
    <button @click="updatePost(post)">Mettre à jour</button>
    <button @click="deletePost(post)">Supprimer</button>

    <p>👍 Likes : {{ post.likes }}</p>
    <button @click="likePost(post)">💚 Like</button>

    <!-- COMMENTAIRES -->
    <div class="comments">
      <h4>💬 Commentaires</h4>

      <ul>
        <li v-for="c in post.comments">{{ c.text }}</li>
      </ul>

      <input v-model="newComment" placeholder="Nouveau commentaire" />
      <button @click="addComment(post)">Ajouter commentaire</button>
    </div>
  </div>
</template>

<style scoped>
.post {
  border: 1px solid #ccc;
  padding: 15px;
  margin-bottom: 15px;
  border-radius: 8px;
}
.comments {
  margin-top: 10px;
  padding: 10px;
  border-left: 3px solid #42b883;
}
button {
  margin-right: 10px;
}
</style>

