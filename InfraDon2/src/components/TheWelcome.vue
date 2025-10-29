<script setup lang="ts">
import { onMounted, ref } from 'vue'
import PouchDB from 'pouchdb'
const counter = ref(0);
const increment = () => {
  counter.value++;
}

const postsData = ref<any[]>([]);


const storage = ref<PouchDB.Database | null>(null);

const initDatabase = () => {
  const db = new PouchDB('http://admin:admin@localhost:5984/post');
  if (db) {
    console.log("Connected to remote collection 'post'");
  } else {
    console.warn("Something went wrong");
  }
  storage.value = db;
}


const fetchData = () => {
  // self = le composant sur je manipule
  const s = storage.value;
  if (s) {
    // function de l'objet de ma base de donnéee
    // donc je ne suis plus dans mon composant
    s.allDocs({
      include_docs: true,
      attachments: true
    }).then((result: any) => {
      console.log('fetchData success =>', result);
      postsData.value = result.rows;
      console.log(result.rows)
    }).catch((error: any) => {
      console.log('fetchData error', error);
    })
  }
}

onMounted(() => {
  initDatabase();
  fetchData();
})

</script>

<template>
  <h1>COUCOUUUUU</h1>
  <p>Counter: {{ counter }}</p>
  <button @click="increment">Increment</button>
  <div v-for="post in postsData" :key="post._id">
    <p>{{ post.doc.post_name }}</p>
    <p>{{ post.doc.post_content }}</p>
  </div>
</template>
