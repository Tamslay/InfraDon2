<script setup lang="ts">
import { ref, onMounted } from "vue"
import PouchDB from "pouchdb"
import PouchFind from "pouchdb-find"

PouchDB.plugin(PouchFind)

const localDB = ref<any>(null)
const remoteDB = ref<any>(null)

const posts = ref<any[]>([])
const searchText = ref("")
const newTitle = ref("")
const newContent = ref("")
const factoryCount = ref(20)

const newComment = ref("")
const editingContent = ref("")
const online = ref(true)

const initDB = () => {
  localDB.value = new PouchDB("posts-local")
  remoteDB.value = new PouchDB("http://admin:matteo@localhost:5984/message")
}

const createIndex = async () => {
  await localDB.value.createIndex({ index: { fields: ["createdAt"] } })
  await localDB.value.createIndex({ index: { fields: ["title"] } })
}

const loadPosts = async () => {
  const result = await localDB.value.find({
    selector: { createdAt: { $gte: null } },
    sort: [{ createdAt: "desc" }]
  })
  posts.value = result.docs
}

const addPost = async () => {
  if (!newTitle.value || !newContent.value) return
  const now = new Date().toISOString()
  const doc = {
    _id: now,
    title: newTitle.value,
    content: newContent.value,
    likes: 0,
    comments: [],
    createdAt: now
  }
  await localDB.value.put(doc)
  newTitle.value = ""
  newContent.value = ""
  loadPosts()
}

const deletePost = async (post: any) => {
  await localDB.value.remove(post)
  loadPosts()
}

const updatePost = async (post: any) => {
  const updated = { ...post }
  await localDB.value.put(updated)
  loadPosts()
}

const likePost = async (post: any) => {
  post.likes++
  await localDB.value.put(post)
  loadPosts()
}

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

const searchPosts = async () => {
  if (!searchText.value) return loadPosts()
  const result = await localDB.value.find({
    selector: {
      title: { $regex: searchText.value },
      createdAt: { $gte: null }
    },
    sort: [{ createdAt: "desc" }]
  })
  posts.value = result.docs
}

const generateFakePosts = async () => {
  const batch = []
  for (let i = 0; i < factoryCount.value; i++) {
    const now = new Date().toISOString()
    batch.push({
      _id: now + "-" + i,
      title: "Fake Post " + i,
      content: "Lorem ipsum " + Math.random(),
      likes: Math.floor(Math.random() * 50),
      comments: [],
      createdAt: now
    })
  }
  await localDB.value.bulkDocs(batch)
  loadPosts()
}

const startSync = () => {
  localDB.value.sync(remoteDB.value, {
    live: true,
    retry: true
  })
    .on("change", loadPosts)
    .on("paused", () => online.value = false)
    .on("active", () => online.value = true)
}

const listenLocalChanges = () => {
  localDB.value.changes({
    since: "now",
    live: true,
    include_docs: true
  }).on("change", loadPosts)
}

onMounted(() => {
  initDB()
  createIndex()
  loadPosts()
  listenLocalChanges()
  startSync()
})
</script>

<template>
  <h1>💬 CouchDB + Vue.js + PouchDB</h1>

  <p :style="{ color: online ? 'green' : 'red' }">
    ● Statut : <strong>{{ online ? "Online" : "Offline" }}</strong>
  </p>

  <h2>Ajouter un post</h2>
  <input v-model="newTitle" placeholder="Titre du post" />
  <input v-model="newContent" placeholder="Contenu du post" />
  <button @click="addPost">Ajouter</button>

  <hr>

  <h2>Rechercher un post</h2>
  <input v-model="searchText" placeholder="Recherche titre" />
  <button @click="searchPosts">Rechercher</button>

  <hr>

  <button @click="startSync">Synchroniser vers CouchDB</button>

  <hr>

  <h2>Générer des posts</h2>
  <input v-model="factoryCount" type="number" />
  <button @click="generateFakePosts">Générer</button>

  <hr>

  <h2>Liste des posts</h2>

  <div v-for="post in posts" :key="post._id" class="post">
    <h3>{{ post.title }}</h3>
    <p>{{ post.content }}</p>

    <input v-model="post.content" />
    <button @click="updatePost(post)">Mettre à jour</button>
    <button @click="deletePost(post)">Supprimer</button>

    <p>Likes : {{ post.likes }}</p>
    <button @click="likePost(post)">Like</button>

    <div class="comments">
      <h4>Commentaires</h4>

      <ul>
        <li v-for="c in post.comments" :key="c.date">{{ c.text }}</li>
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
  border-radius: 5px;
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
