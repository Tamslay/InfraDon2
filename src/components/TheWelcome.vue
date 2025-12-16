<script setup lang="ts">
import { ref, onMounted } from "vue"
import PouchDB from "pouchdb"
import PouchFind from "pouchdb-find"

PouchDB.plugin(PouchFind)

const postsDB = ref<any>(null)
const commentsDB = ref<any>(null)
const postsRemote = ref<any>(null)
const commentsRemote = ref<any>(null)

const posts = ref<any[]>([])
const comments = ref<Record<string, any[]>>({})

const newTitle = ref("")
const newContent = ref("")
const searchText = ref("")
const newComment = ref<Record<string, string>>({})
const editComment = ref<Record<string, string>>({})
const editingCommentId = ref<string | null>(null)
const factoryCount = ref<number>(20)
const online = ref(true)

const initDB = () => {
  postsDB.value = new PouchDB("posts-local")
  commentsDB.value = new PouchDB("comments-local")
  postsRemote.value = new PouchDB("http://admin:matteo@localhost:5984/posts")
  commentsRemote.value = new PouchDB("http://admin:matteo@localhost:5984/comments")
}

const createIndexes = async () => {
  await postsDB.value.createIndex({ index: { fields: ["createdAt"] } })
  await postsDB.value.createIndex({ index: { fields: ["title"] } })
  await commentsDB.value.createIndex({ index: { fields: ["postId"] } })
}

const loadPosts = async () => {
  const result = await postsDB.value.find({
    selector: { createdAt: { $gte: null } },
    sort: [{ createdAt: "desc" }]
  })
  posts.value = result.docs
}

const loadCommentsForPost = async (postId: string) => {
  const result = await commentsDB.value.find({
    selector: { postId }
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
}

const startEditComment = (comment: any) => {
  editingCommentId.value = comment._id
  editComment.value[comment._id] = comment.text
}

const updateComment = async (comment: any) => {
  await commentsDB.value.put({
    ...comment,
    text: editComment.value[comment._id]
  })
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
    batch.push({
      _id: crypto.randomUUID(),
      title: "Fake Post " + i,
      content: "Lorem ipsum " + Math.random(),
      likes: Math.floor(Math.random() * 50),
      createdAt: new Date().toISOString()
    })
  }
  await postsDB.value.bulkDocs(batch)
  await loadPosts()
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
  <input v-model="searchText" placeholder="Titre" />
  <button @click="searchPosts">Rechercher</button>

  <hr>

  <h2>Factory</h2>
  <input v-model.number="factoryCount" type="number" />
  <button @click="generateFakePosts">Générer</button>

  <hr>

  <h2>Posts</h2>

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
      <button @click="loadCommentsForPost(post._id)">Charger</button>

      <ul>
        <li v-for="c in comments[post._id]" :key="c._id">
          <span v-if="editingCommentId !== c._id">{{ c.text }}</span>
          <input
            v-else
            v-model="editComment[c._id]"
          />
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
  border-radius: 5px;
}
.comments {
  margin-top: 10px;
  padding: 10px;
  border-left: 3px solid #42b883;
}
button {
  margin-right: 5px;
}
</style>
