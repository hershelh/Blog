<script setup lang="ts">
useHead({
  meta: [
    { property: 'og:title', content: 'Hershel Hahn' },
    { property: 'og:image', content: 'https://antfu.me/og-icon.png' },
    { name: 'description', content: 'Hershel Hahn\'s Portfolio' },
  ],
})
</script>

<template>
  <NavBar />
  <main class="px-7 py-10">
    <router-view />
    <Footer />
  </main>
</template>
