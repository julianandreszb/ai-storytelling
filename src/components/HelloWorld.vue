<script setup lang="ts">
defineProps<{
  msg: string
}>()

const CLOUDFLARE_WORKERS_AI_TOKEN = import.meta.env.VITE_CLOUDFLARE_WORKERS_AI_TOKEN

console.log(CLOUDFLARE_WORKERS_AI_TOKEN)

fetch("https://api.cloudflare.com/client/v4/user/tokens/verify", {
  method: "GET",
  headers: {
    "Authorization": `Bearer ${CLOUDFLARE_WORKERS_AI_TOKEN}`,
    "Content-Type": "application/json"
  }
})
  .then(response => response.json())
  .then(data => console.log(data))
  .catch((error) => console.error('Error:', error));

</script>

<template>
  <div class="greetings">
    <h1 class="green">{{ msg }}</h1>
  </div>
</template>

<style scoped>
h1 {
  font-weight: 500;
  font-size: 2.6rem;
  position: relative;
  top: -10px;
}

h3 {
  font-size: 1.2rem;
}

.greetings h1,
.greetings h3 {
  text-align: center;
}

@media (min-width: 1024px) {
  .greetings h1,
  .greetings h3 {
    text-align: left;
  }
}
</style>
