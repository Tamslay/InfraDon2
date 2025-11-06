<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'

// ---------------------
// 🔹 VARIABLES RÉACTIVES
// ---------------------
const counter = ref(0)
const postsData = ref<any[]>([])
const storage = ref<PouchDB.Database | null>(null)

// Champs pour le formulaire
const newTitle = ref('')
const newContent = ref('')

// ---------------------
// 🔹 FONCTIONS DE BASE
// ---------------------
const increment = () => {
  counter.value++
}

// Connexion à CouchDB
const initDatabase = () => {
  const db = new PouchDB('http://admin:matteo@localhost:5984/message')
  if (db) {
    console.log("✅ Connecté à la base 'message'")
  } else {
    console.warn("⚠️ Erreur de connexion à la base CouchDB")
  }
  storage.value = db
}

// Récupérer toutes les données
const fetchData = () => {
  const s = storage.value
  if (s) {
    s.allDocs({
      include_docs: true,
      attachments: true
    })
      .then((result: any) => {
        console.log('✅ fetchData success =>', result)
        postsData.value = result.rows
      })
      .catch((error: any) => {
        console.log('❌ fetchData error', error)
      })
  }
}

// ---------------------
// 🔹 AJOUTER UN DOCUMENT
// ---------------------
const addPost = async (postName: string, postContent: string) => {
  if (!storage.value) return
  try {
    const response = await storage.value.post({
      post_name: postName,
      post_content: postContent,
      created_at: new Date().toISOString()
    })
    console.log('✅ Document ajouté :', response)
    newTitle.value = ''
    newContent.value = ''
    fetchData()
  } catch (err) {
    console.error('❌ Erreur ajout doc:', err)
  }
}

// ---------------------
// 🔹 SUPPRIMER UN DOCUMENT
// ---------------------
const deletePost = async (id: string, rev: string) => {
  if (!storage.value) return
  try {
    const response = await storage.value.remove(id, rev)
    console.log('🗑️ Document supprimé :', response)
    fetchData()
  } catch (err) {
    console.error('❌ Erreur suppression doc:', err)
  }
}

// ---------------------
// 🔹 METTRE À JOUR UN DOCUMENT
// ---------------------
const updatePost = async (post: any, newContent: string) => {
  if (!storage.value) return
  try {
    const updatedDoc = { ...post.doc, post_content: newContent }
    const response = await storage.value.put(updatedDoc)
    console.log('✏️ Document mis à jour :', response)
    fetchData()
  } catch (err) {
    console.error('❌ Erreur mise à jour doc:', err)
  }
}

// ---------------------
// 🔹 SYNCHRONISATION TEMPS RÉEL
// ---------------------
const syncWithRemote = () => {
  if (!storage.value) return
  const remote = new PouchDB('http://admin:matteo@localhost:5984/message')
  storage.value.sync(remote, {
    live: true,
    retry: true
  })
    .on('change', info => {
      console.log('🔄 Changement détecté', info)
      fetchData()
    })
    .on('error', err => {
      console.error('❌ Erreur sync:', err)
    })
}

// ---------------------
// 🔹 AU MONTAGE DU COMPOSANT
// ---------------------
onMounted(() => {
  initDatabase()
  fetchData()
  syncWithRemote()
})
</script>

<template>
  <h1>💬 CouchDB + Vue.js + PouchDB</h1>

  <p>Counter: {{ counter }}</p>
  <button @click="increment">Increment</button>

  <hr>

  <h2>➕ Ajouter un post</h2>
  <input v-model="newTitle" placeholder="Titre du post" />
  <input v-model="newContent" placeholder="Contenu du post" />
  <button @click="addPost(newTitle, newContent)">Ajouter</button>

  <hr>

  <h2>📚 Liste des posts</h2>
  <div v-for="post in postsData" :key="post.id" style="border:1px solid #ccc; padding:10px; margin-bottom:10px;">
    <p><strong>{{ post.doc.post_name }}</strong></p>
    <p>{{ post.doc.post_content }}</p>

    <input
      v-model="post.doc.post_content"
      placeholder="Modifier le contenu"
      style="width: 100%;"
    />
    <button @click="updatePost(post, post.doc.post_content)">Mettre à jour</button>
    <button @click="deletePost(post.id, post.value.rev)">Supprimer</button>
  </div>
</template>

<style scoped>
.container {
  padding: 1.5rem;
  color: white;
  max-width: 720px;
  margin: auto;
}
.actions {
  margin-bottom: 1rem;
}
.form {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  margin-bottom: 1.5rem;
}
input {
  padding: 0.5rem;
  border-radius: 4px;
  border: none;
}
button {
  background: #42b883;
  color: white;
  padding: 0.5rem;
  border: none;
  cursor: pointer;
  border-radius: 4px;
}
button:hover {
  background: #2a9d6e;
}
.item {
  background: #1e1e1e;
  padding: 1rem;
  margin-top: 0.75rem;
  border-radius: 6px;
}
.row {
  display: flex;
  gap: 0.5rem;
  margin-top: 0.75rem;
}
.editBox {
  padding: 1rem;
  border: 2px solid #42b883;
  border-radius: 8px;
  background: #17221f;
}
</style>
 


