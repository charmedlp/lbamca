<script setup lang="ts">
const props = defineProps<{
  gameName: string;
}>();
</script>

<template>
  <div class="music-box">
    <h3>{{ gameName }}</h3>
  </div>
</template>

<style scoped>
h3 {
  color: #ffb619;
  font-size: 8cqi;
  line-height: 1.1;
  margin: 0;
  text-transform: uppercase;
  text-shadow:
    1px 1px 5px #000000,
    1px 1px 12px #000000;
}

/* .music-box {
  background-image: url("../assets/images/MusicBox.png");
  background-size: contain;
  background-repeat: no-repeat;
  width: 100%;
  aspect-ratio: 826/300;
  padding: 7%;
  box-sizing: border-box;
  display: flex;
  justify-content: center;
  align-items: center;
  position: relative;
  container-type: inline-size;
} */

.newGame {
  background-image: url("../assets/images/MusicBoxV2-new.png");
}

.matchmaker {
  padding: 4px;
}

.matchmaker h3 {
  font-size: 0.75rem;
}
</style>
