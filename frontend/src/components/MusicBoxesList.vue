<script setup lang="ts">
import MusicBox from "./MusicBox.vue";
import { useGameStore } from "../stores/game";
import { ref } from "vue";

const gameStore = useGameStore();

const props = defineProps<{
  controlPanel?: boolean;
  musicBoxes?: number[];
}>();
</script>

<template>
  <div class="music-boxes-container">
    <div class="music-boxes-content" :class="{ 'game-presentation': !controlPanel }">
      <MusicBox
        v-for="(game, i) in musicBoxes"
        :key="`box-${i}-${game}`"
        :gameName="gameStore.musicBoxesGames.find((g) => g.idGame === game)?.gameName || ''"
        class="music-box"
      ></MusicBox>
    </div>
  </div>
</template>

<style scoped>
.music-boxes-container {
  padding: 1rem;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  width: 600px;
  max-width: 100%;
}

.music-boxes-content {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr 1fr;
  gap: 0.5rem;
}

/* .music-boxes-content.game-presentation {
  padding-top: 4rem;
} */

.music-box {
  cursor: pointer;
}

.slide-up-enter-active,
.slide-up-leave-active {
  opacity: 1;
  transform: translateY(0vh);
  transition:
    transform 1s ease-in-out,
    opacity 0.25s ease-in-out;
}

.slide-up-enter-from,
.slide-up-leave-to {
  opacity: 0;
  transform: translateY(100vh);
}
</style>
