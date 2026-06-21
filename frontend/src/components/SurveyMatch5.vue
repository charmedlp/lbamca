<script setup lang="ts">
import { computed, onMounted, ref } from "vue";
import MusicBoxesList from "./MusicBoxesList.vue";
import MusicBox from "./MusicBox.vue";
import { useGameStore } from "../stores/game";
const gameStore = useGameStore();

function onBeforeLeave(el: Element) {
  const htmlEl = el as HTMLElement;
  htmlEl.style.top = htmlEl.offsetTop + "px";
}

onMounted(() => {
  gameStore.musicBoxesGames.sort((a, b) => Math.min(...a.matchesPlayed) - Math.min(...b.matchesPlayed));
});

// function putGameInMusicBoxes(gameId: number, position: number, type: number) {
//   console.log(type, position, gameId);
//   if (position == 0) {
//     gameStore.selectedMusicBoxes[4 * (position - 1) + (type - 1)] = null;
//   } else if (gameStore.selectedMusicBoxes.includes(gameId)) {
//     // TODO: remove game from other positions
//     gameStore.selectedMusicBoxes = gameStore.selectedMusicBoxes.map((game) => {
//       if (game === gameId) {
//         return null;
//       }
//       return game;
//     });
//     gameStore.selectedMusicBoxes[4 * (position - 1) + (type - 1)] = gameId;
//   } else {
//     gameStore.selectedMusicBoxes[4 * (position - 1) + (type - 1)] = gameId;
//   }
// }

// Drag & drop
const dragState = ref<{ gameId: number; games: any[]; maxPositions: number; gameType?: number; fromSelected: boolean } | null>(null);
const dragOverSlot = ref<{ games: any[]; position: number; gameType?: number } | null>(null);
const dropHandled = ref(false);

function onDragStart(games: any[], gameId: number, maxPositions: number, event: DragEvent, fromSelected = false) {
  const game = games.find((g) => g.idGame === gameId);
  dragState.value = { gameId, games, maxPositions, gameType: game?.type, fromSelected };
  dropHandled.value = false;
  event.dataTransfer!.effectAllowed = "move";
}

function onDragEnd() {
  if (dragState.value && dragState.value.fromSelected && !dropHandled.value) {
    applyPositionChange(dragState.value.games, dragState.value.gameId, null, dragState.value.maxPositions);
  }
  dragState.value = null;
  dragOverSlot.value = null;
  dropHandled.value = false;
}

function onSlotDragOver(games: any[], position: number, event: DragEvent, gameType?: number) {
  if (!dragState.value || dragState.value.games !== games) return;
  if (gameType != null && dragState.value.gameType !== gameType) return;
  event.preventDefault();
  dragOverSlot.value = { games, position, gameType };
}

function onSlotDragLeave() {
  dragOverSlot.value = null;
}

function onSlotDrop(games: any[], position: number, maxPositions: number, gameType?: number) {
  if (!dragState.value || dragState.value.games !== games) return;
  if (gameType != null && dragState.value.gameType !== gameType) return;
  dropHandled.value = true;
  applyPositionChange(games, dragState.value.gameId, position, maxPositions);
  dragState.value = null;
  dragOverSlot.value = null;
}

function isDragOver(games: any[], position: number, gameType?: number) {
  if (!dragOverSlot.value) return false;
  if (dragOverSlot.value.games !== games || dragOverSlot.value.position !== position) return false;
  if (gameType != null && dragOverSlot.value.gameType !== gameType) return false;
  return true;
}

function applyPositionChange(games: any[], gameId: number, newPosition: number | null, maxPositions: number) {
  const game = games.find((g) => g.idGame === gameId);
  if (!game) return;

  const oldPosition = game.position;
  const gameType = game.type;
  const isSameGroup = (g: any) => g.idGame != gameId && (gameType != null ? g.type == gameType : true);

  if (newPosition === oldPosition) return;

  // Suppression: décaler les boîtes suivantes vers le haut
  if (newPosition == null) {
    game.position = null;
    for (let i = (oldPosition ?? 0) + 1; i <= maxPositions; i++) {
      const subsequentGame = games.find((g) => isSameGroup(g) && g.position == i);
      if (subsequentGame) subsequentGame.position = i - 1;
    }
    return;
  }

  // Ajout (ancienne position nulle): décaler les boîtes vers le bas
  if (oldPosition == null) {
    game.position = newPosition;
    let lastReplacedGameId: number | null = null;
    for (let i = newPosition; i <= maxPositions; i++) {
      const subsequentGame = games.find((g) => isSameGroup(g) && g.position == i && g.idGame != lastReplacedGameId);
      if (subsequentGame) {
        subsequentGame.position = i >= maxPositions ? null : i + 1;
        lastReplacedGameId = subsequentGame.idGame;
      } else {
        break;
      }
    }
    return;
  }

  // Si la position cible est libre, simplement déplacer sans décaler
  if (!games.find((g) => isSameGroup(g) && g.position == newPosition)) {
    game.position = newPosition;
    return;
  }

  // Déplacement vers le haut (ex: de 4 à 2): décaler les boîtes entre les deux vers le bas
  if (newPosition < oldPosition) {
    for (let i = oldPosition - 1; i >= newPosition; i--) {
      const gameAtPosition = games.find((g) => isSameGroup(g) && g.position == i);
      if (gameAtPosition) gameAtPosition.position = i + 1;
    }
    game.position = newPosition;
    return;
  }

  // Déplacement vers le bas (ex: de 2 à 4): décaler les boîtes entre les deux vers le haut
  if (newPosition > oldPosition) {
    for (let i = oldPosition + 1; i <= newPosition; i++) {
      const gameAtPosition = games.find((g) => isSameGroup(g) && g.position == i);
      if (gameAtPosition) gameAtPosition.position = i - 1;
    }
    game.position = newPosition;
    return;
  }
}

function updateGamePositions(games: any[], gameId: number, event: Event, maxPositions: number) {
  const target = event.target as HTMLSelectElement;
  const newPosition = target.value === "" ? null : parseInt(target.value);
  applyPositionChange(games, gameId, newPosition, maxPositions);
}

// Computed properties pour les jeux sélectionnés par type, triés par position
const selectedTransformedGames = computed(() => {
  return gameStore.musicBoxesGames.filter((game) => game.type === 3 && game.position !== null).sort((a, b) => (a.position ?? 0) - (b.position ?? 0));
});

const selectedTextualGames = computed(() => {
  return gameStore.musicBoxesGames.filter((game) => game.type === 2 && game.position !== null).sort((a, b) => (a.position ?? 0) - (b.position ?? 0));
});

const selectedSpecialGames = computed(() => {
  return gameStore.musicBoxesGames.filter((game) => game.type === 1 && game.position !== null).sort((a, b) => (a.position ?? 0) - (b.position ?? 0));
});

const musicBoxes = computed(() => {
  return [
    gameStore.musicBoxesGames.find((game) => game.type == 1 && game.position == 1)?.idGame ?? null,
    gameStore.musicBoxesGames.find((game) => game.type == 2 && game.position == 1)?.idGame ?? null,
    gameStore.musicBoxesGames.find((game) => game.type == 3 && game.position == 1)?.idGame ?? null,
    39,
    gameStore.musicBoxesGames.find((game) => game.type == 1 && game.position == 2)?.idGame ?? null,
    gameStore.musicBoxesGames.find((game) => game.type == 2 && game.position == 2)?.idGame ?? null,
    gameStore.musicBoxesGames.find((game) => game.type == 3 && game.position == 2)?.idGame ?? null,
    5,
    gameStore.musicBoxesGames.find((game) => game.type == 1 && game.position == 3)?.idGame ?? null,
    gameStore.musicBoxesGames.find((game) => game.type == 2 && game.position == 3)?.idGame ?? null,
    gameStore.musicBoxesGames.find((game) => game.type == 3 && game.position == 3)?.idGame ?? null,
    5,
    gameStore.musicBoxesGames.find((game) => game.type == 1 && game.position == 4)?.idGame ?? null,
    gameStore.musicBoxesGames.find((game) => game.type == 2 && game.position == 4)?.idGame ?? null,
    gameStore.musicBoxesGames.find((game) => game.type == 3 && game.position == 4)?.idGame ?? null,
    5,
    gameStore.musicBoxesGames.find((game) => game.type == 1 && game.position == 5)?.idGame ?? null,
    gameStore.musicBoxesGames.find((game) => game.type == 2 && game.position == 5)?.idGame ?? null,
    gameStore.musicBoxesGames.find((game) => game.type == 3 && game.position == 5)?.idGame ?? null,
    5,
  ];
});

const completed = computed(
  () =>
    gameStore.musicBoxesGames.filter((game) => game.position != null).length == 15 &&
    gameStore.guessingGames.filter((game) => game.position != null && game.idGame != null).length == 3 &&
    gameStore.singingGames.filter((game) => game.position != null && game.idGame == null).length == 3 &&
    gameStore.specialGames.filter((game) => game.position != null && game.idGame != null).length == 3,
);
</script>

<template>
  <div class="survey-container">
    <div class="survey-content">
      <div class="survey-header">
        <h1>La Boîte à Musique V</h1>
        <h4>VOUS choisissez les jeux qui composeront ce match!</h4>
      </div>
      <div class="music-boxes-container-preview">
        <h2>Les Boîtes à Musique</h2>
        <MusicBoxesList :musicBoxes="musicBoxes" :controlPanel="false"></MusicBoxesList>
      </div>
      <div class="survey-questions">
        <div class="survey-question">
          <div class="question-header">
            <h3>Jeux de chansons transformées</h3>
            <label>
              Sélectionnez cinq jeux qui feront partie de votre tableau des Boîtes à Musique, en les plaçant de votre préféré (1) au moins préféré (5).
            </label>
          </div>
          <div class="question-elements">
            <div class="question-options">
              <div class="music-box-element" v-for="game in gameStore.musicBoxesGames.filter((g) => g.type == 3)">
                <MusicBox
                  :key="game.idGame"
                  :gameName="game.gameName"
                  draggable="true"
                  @dragstart="onDragStart(gameStore.musicBoxesGames, game.idGame, 5, $event)"
                  @dragend="onDragEnd"
                >
                </MusicBox>
                <select :value="game.position" name="" id="" @change="updateGamePositions(gameStore.musicBoxesGames, game.idGame, $event, 5)">
                  <option :value="null"></option>
                  <option :value="1">1</option>
                  <option :value="2">2</option>
                  <option :value="3">3</option>
                  <option :value="4">4</option>
                  <option :value="5">5</option>
                </select>
                <div class="game-description">
                  <label for="">
                    Match{{ game.matchesPlayed.length > 1 ? "s" : "" }}
                    {{ new Intl.ListFormat("fr-CA", { style: "long", type: "conjunction" }).format(game.matchesPlayed.map(String)) }}
                  </label>
                  <p>{{ game.description }}</p>
                </div>
              </div>
            </div>
            <div class="vertical-line"></div>
            <div class="question-selected">
              <div class="position-numbers">
                <h2 v-for="i in 5" :key="i">{{ i }}</h2>
              </div>
              <TransitionGroup name="music-box-list" tag="div" class="question-selected-wrapper" @before-leave="onBeforeLeave">
                <div
                  v-for="i in 5"
                  :key="'box-' + (selectedTransformedGames.find((game) => game.position == i)?.idGame ?? 'empty-' + i)"
                  class="selected-music-box"
                  :class="{ 'drag-over': isDragOver(gameStore.musicBoxesGames, i, 3) }"
                  :draggable="!!selectedTransformedGames.find((game) => game.position == i)"
                  @dragstart="onDragStart(gameStore.musicBoxesGames, selectedTransformedGames.find((game) => game.position == i)?.idGame, 5, $event, true)"
                  @dragend="onDragEnd"
                  @dragover="onSlotDragOver(gameStore.musicBoxesGames, i, $event, 3)"
                  @dragleave="onSlotDragLeave"
                  @drop="onSlotDrop(gameStore.musicBoxesGames, i, 5, 3)"
                >
                  <MusicBox
                    :class="{ invisible: !selectedTransformedGames.find((game) => game.position == i) }"
                    :gameName="selectedTransformedGames.find((game) => game.position == i)?.gameName ?? ''"
                  ></MusicBox>
                </div>
              </TransitionGroup>
            </div>
          </div>
        </div>
        <div class="survey-question">
          <div class="question-header">
            <h3>Jeux textuels/de paroles</h3>
            <label>
              Sélectionnez cinq jeux qui feront partie de votre tableau des Boîtes à Musique, en les plaçant de votre préféré (1) au moins préféré (5).
            </label>
          </div>
          <div class="question-elements">
            <div class="question-options">
              <div class="music-box-element" v-for="game in gameStore.musicBoxesGames.filter((g) => g.type == 2)">
                <MusicBox
                  :key="game.idGame"
                  :gameName="game.gameName"
                  draggable="true"
                  @dragstart="onDragStart(gameStore.musicBoxesGames, game.idGame, 5, $event)"
                  @dragend="onDragEnd"
                >
                </MusicBox>
                <select :value="game.position" name="" id="" @change="updateGamePositions(gameStore.musicBoxesGames, game.idGame, $event, 5)">
                  <option :value="null"></option>
                  <option :value="1">1</option>
                  <option :value="2">2</option>
                  <option :value="3">3</option>
                  <option :value="4">4</option>
                  <option :value="5">5</option>
                </select>
                <div class="game-description">
                  <label for="">
                    Match{{ game.matchesPlayed.length > 1 ? "s" : "" }}
                    {{ new Intl.ListFormat("fr-CA", { style: "long", type: "conjunction" }).format(game.matchesPlayed.map(String)) }}
                  </label>
                  <p>{{ game.description }}</p>
                </div>
              </div>
            </div>
            <div class="vertical-line"></div>
            <div class="question-selected">
              <div class="position-numbers">
                <h2 v-for="i in 5" :key="i">{{ i }}</h2>
              </div>
              <TransitionGroup name="music-box-list" tag="div" class="question-selected-wrapper" @before-leave="onBeforeLeave">
                <div
                  v-for="i in 5"
                  :key="'box-' + (selectedTextualGames.find((game) => game.position == i)?.idGame ?? 'empty-' + i)"
                  class="selected-music-box"
                  :class="{ 'drag-over': isDragOver(gameStore.musicBoxesGames, i, 2) }"
                  :draggable="!!selectedTextualGames.find((game) => game.position == i)"
                  @dragstart="onDragStart(gameStore.musicBoxesGames, selectedTextualGames.find((game) => game.position == i)?.idGame, 5, $event, true)"
                  @dragend="onDragEnd"
                  @dragover="onSlotDragOver(gameStore.musicBoxesGames, i, $event, 2)"
                  @dragleave="onSlotDragLeave"
                  @drop="onSlotDrop(gameStore.musicBoxesGames, i, 5, 2)"
                >
                  <MusicBox
                    :class="{ invisible: !selectedTextualGames.find((game) => game.position == i) }"
                    :gameName="selectedTextualGames.find((game) => game.position == i)?.gameName ?? ''"
                  ></MusicBox>
                </div>
              </TransitionGroup>
            </div>
          </div>
        </div>
        <div class="survey-question">
          <div class="question-header">
            <h3>Jeux spéciaux/visuels</h3>
            <label>
              Sélectionnez cinq jeux qui feront partie de votre tableau des Boîtes à Musique, en les plaçant de votre préféré (1) au moins préféré (5).
            </label>
          </div>
          <div class="question-elements">
            <div class="question-options">
              <div class="music-box-element" v-for="game in gameStore.musicBoxesGames.filter((g) => g.type == 1)">
                <MusicBox
                  :key="game.idGame"
                  :gameName="game.gameName"
                  draggable="true"
                  @dragstart="onDragStart(gameStore.musicBoxesGames, game.idGame, 5, $event)"
                  @dragend="onDragEnd"
                >
                </MusicBox>
                <select :value="game.position" name="" id="" @change="updateGamePositions(gameStore.musicBoxesGames, game.idGame, $event, 5)">
                  <option :value="null"></option>
                  <option :value="1">1</option>
                  <option :value="2">2</option>
                  <option :value="3">3</option>
                  <option :value="4">4</option>
                  <option :value="5">5</option>
                </select>
                <div class="game-description">
                  <label for="">
                    Match{{ game.matchesPlayed.length > 1 ? "s" : "" }}
                    {{ new Intl.ListFormat("fr-CA", { style: "long", type: "conjunction" }).format(game.matchesPlayed.map(String)) }}
                  </label>
                  <p>{{ game.description }}</p>
                </div>
              </div>
            </div>
            <div class="vertical-line"></div>
            <div class="question-selected">
              <div class="position-numbers">
                <h2 v-for="i in 5" :key="i">{{ i }}</h2>
              </div>
              <TransitionGroup name="music-box-list" tag="div" class="question-selected-wrapper" @before-leave="onBeforeLeave">
                <div
                  v-for="i in 5"
                  :key="'box-' + (selectedSpecialGames.find((game) => game.position == i)?.idGame ?? 'empty-' + i)"
                  class="selected-music-box"
                  :class="{ 'drag-over': isDragOver(gameStore.musicBoxesGames, i, 1) }"
                  :draggable="!!selectedSpecialGames.find((game) => game.position == i)"
                  @dragstart="onDragStart(gameStore.musicBoxesGames, selectedSpecialGames.find((game) => game.position == i)?.idGame, 5, $event, true)"
                  @dragend="onDragEnd"
                  @dragover="onSlotDragOver(gameStore.musicBoxesGames, i, $event, 1)"
                  @dragleave="onSlotDragLeave"
                  @drop="onSlotDrop(gameStore.musicBoxesGames, i, 5, 1)"
                >
                  <MusicBox
                    :class="{ invisible: !selectedSpecialGames.find((game) => game.position == i) }"
                    :gameName="selectedSpecialGames.find((game) => game.position == i)?.gameName ?? ''"
                  ></MusicBox>
                </div>
              </TransitionGroup>
            </div>
          </div>
        </div>
        <div class="survey-question">
          <div class="question-header">
            <h2>Jeux pour faire deviner</h2>
            <label> Sélectionnez vos trois jeux favoris, en les plaçant de votre préféré (1) au moins préféré (3). </label>
          </div>
          <div class="question-elements">
            <div class="question-options">
              <div class="music-box-element" v-for="game in gameStore.guessingGames">
                <MusicBox
                  :key="game.idGame"
                  :gameName="game.gameName"
                  draggable="true"
                  @dragstart="onDragStart(gameStore.guessingGames, game.idGame, 3, $event)"
                  @dragend="onDragEnd"
                >
                </MusicBox>
                <select :value="game.position" name="" id="" @change="updateGamePositions(gameStore.guessingGames, game.idGame, $event, 3)">
                  <option :value="null"></option>
                  <option :value="1">1</option>
                  <option :value="2">2</option>
                  <option :value="3">3</option>
                </select>
                <div class="game-description">
                  <label for="">
                    Match{{ game.matchesPlayed.length > 1 ? "s" : "" }}
                    {{ new Intl.ListFormat("fr-CA", { style: "long", type: "conjunction" }).format(game.matchesPlayed.map(String)) }}
                  </label>
                  <p>{{ game.description }}</p>
                </div>
              </div>
            </div>
            <div class="vertical-line"></div>
            <div class="question-selected">
              <div class="position-numbers">
                <h2 v-for="i in 3" :key="i">{{ i }}</h2>
              </div>
              <TransitionGroup name="music-box-list" tag="div" class="question-selected-wrapper" @before-leave="onBeforeLeave">
                <div
                  v-for="i in 3"
                  :key="'box-' + (gameStore.guessingGames.find((game) => game.position == i)?.idGame ?? 'empty-' + i)"
                  class="selected-music-box"
                  :class="{ 'drag-over': isDragOver(gameStore.guessingGames, i) }"
                  :draggable="!!gameStore.guessingGames.find((game) => game.position == i)"
                  @dragstart="onDragStart(gameStore.guessingGames, gameStore.guessingGames.find((game) => game.position == i)?.idGame, 3, $event, true)"
                  @dragend="onDragEnd"
                  @dragover="onSlotDragOver(gameStore.guessingGames, i, $event)"
                  @dragleave="onSlotDragLeave"
                  @drop="onSlotDrop(gameStore.guessingGames, i, 3)"
                >
                  <MusicBox
                    :class="{ invisible: !gameStore.guessingGames.find((game) => game.position == i) }"
                    :gameName="gameStore.guessingGames.find((game) => game.position == i)?.gameName ?? ''"
                  ></MusicBox>
                </div>
              </TransitionGroup>
            </div>
          </div>
        </div>
        <div class="survey-question">
          <div class="question-header">
            <h2>Jeux d'énumération</h2>
            <label> Sélectionnez vos trois jeux favoris, en les plaçant de votre préféré (1) au moins préféré (3). </label>
          </div>
          <div class="question-elements">
            <div class="question-options">
              <div class="music-box-element" v-for="game in gameStore.namingGames">
                <MusicBox
                  :key="game.idGame"
                  :gameName="game.gameName"
                  draggable="true"
                  @dragstart="onDragStart(gameStore.namingGames, game.idGame, 3, $event)"
                  @dragend="onDragEnd"
                >
                </MusicBox>
                <select :value="game.position" name="" id="" @change="updateGamePositions(gameStore.namingGames, game.idGame, $event, 3)">
                  <option :value="null"></option>
                  <option :value="1">1</option>
                  <option :value="2">2</option>
                  <option :value="3">3</option>
                </select>
                <div class="game-description">
                  <label for="">
                    Match{{ game.matchesPlayed.length > 1 ? "s" : "" }}
                    {{ new Intl.ListFormat("fr-CA", { style: "long", type: "conjunction" }).format(game.matchesPlayed.map(String)) }}
                  </label>
                  <p>{{ game.description }}</p>
                </div>
              </div>
            </div>
            <div class="vertical-line"></div>
            <div class="question-selected">
              <div class="position-numbers">
                <h2 v-for="i in 3" :key="i">{{ i }}</h2>
              </div>
              <TransitionGroup name="music-box-list" tag="div" class="question-selected-wrapper" @before-leave="onBeforeLeave">
                <div
                  v-for="i in 3"
                  :key="'box-' + (gameStore.namingGames.find((game) => game.position == i)?.idGame ?? 'empty-' + i)"
                  class="selected-music-box"
                  :class="{ 'drag-over': isDragOver(gameStore.namingGames, i) }"
                  :draggable="!!gameStore.namingGames.find((game) => game.position == i)"
                  @dragstart="onDragStart(gameStore.namingGames, gameStore.namingGames.find((game) => game.position == i)?.idGame, 3, $event, true)"
                  @dragend="onDragEnd"
                  @dragover="onSlotDragOver(gameStore.namingGames, i, $event)"
                  @dragleave="onSlotDragLeave"
                  @drop="onSlotDrop(gameStore.namingGames, i, 3)"
                >
                  <MusicBox
                    :class="{ invisible: !gameStore.namingGames.find((game) => game.position == i) }"
                    :gameName="gameStore.namingGames.find((game) => game.position == i)?.gameName ?? ''"
                  ></MusicBox>
                </div>
              </TransitionGroup>
            </div>
          </div>
        </div>
        <div class="survey-question">
          <div class="question-header">
            <h2>Jeux de chant</h2>
            <label> Sélectionnez vos trois jeux favoris, en les plaçant de votre préféré (1) au moins préféré (3). </label>
          </div>
          <div class="question-elements">
            <div class="question-options">
              <div class="music-box-element" v-for="game in gameStore.singingGames">
                <MusicBox
                  :key="game.idGame"
                  :gameName="game.gameName"
                  draggable="true"
                  @dragstart="onDragStart(gameStore.singingGames, game.idGame, 3, $event)"
                  @dragend="onDragEnd"
                >
                </MusicBox>
                <select :value="game.position" name="" id="" @change="updateGamePositions(gameStore.singingGames, game.idGame, $event, 3)">
                  <option :value="null"></option>
                  <option :value="1">1</option>
                  <option :value="2">2</option>
                  <option :value="3">3</option>
                </select>
                <div class="game-description">
                  <label for="">
                    Match{{ game.matchesPlayed.length > 1 ? "s" : "" }}
                    {{ new Intl.ListFormat("fr-CA", { style: "long", type: "conjunction" }).format(game.matchesPlayed.map(String)) }}
                  </label>
                  <p>{{ game.description }}</p>
                </div>
              </div>
            </div>
            <div class="vertical-line"></div>
            <div class="question-selected">
              <div class="position-numbers">
                <h2 v-for="i in 3" :key="i">{{ i }}</h2>
              </div>
              <TransitionGroup name="music-box-list" tag="div" class="question-selected-wrapper" @before-leave="onBeforeLeave">
                <div
                  v-for="i in 3"
                  :key="'box-' + (gameStore.singingGames.find((game) => game.position == i)?.idGame ?? 'empty-' + i)"
                  class="selected-music-box"
                  :class="{ 'drag-over': isDragOver(gameStore.singingGames, i) }"
                  :draggable="!!gameStore.singingGames.find((game) => game.position == i)"
                  @dragstart="onDragStart(gameStore.singingGames, gameStore.singingGames.find((game) => game.position == i)?.idGame, 3, $event, true)"
                  @dragend="onDragEnd"
                  @dragover="onSlotDragOver(gameStore.singingGames, i, $event)"
                  @dragleave="onSlotDragLeave"
                  @drop="onSlotDrop(gameStore.singingGames, i, 3)"
                >
                  <MusicBox
                    :class="{ invisible: !gameStore.singingGames.find((game) => game.position == i) }"
                    :gameName="gameStore.singingGames.find((game) => game.position == i)?.gameName ?? ''"
                  ></MusicBox>
                </div>
              </TransitionGroup>
            </div>
          </div>
        </div>
        <div class="survey-question">
          <div class="question-header">
            <h2>Jeux spéciaux</h2>
            <label> Sélectionnez vos trois jeux favoris, en les plaçant de votre préféré (1) au moins préféré (3). </label>
          </div>
          <div class="question-elements">
            <div class="question-options">
              <div class="music-box-element" v-for="game in gameStore.specialGames">
                <MusicBox
                  :key="game.idGame"
                  :gameName="game.gameName"
                  draggable="true"
                  @dragstart="onDragStart(gameStore.specialGames, game.idGame, 3, $event)"
                  @dragend="onDragEnd"
                >
                </MusicBox>
                <select :value="game.position" name="" id="" @change="updateGamePositions(gameStore.specialGames, game.idGame, $event, 3)">
                  <option :value="null"></option>
                  <option :value="1">1</option>
                  <option :value="2">2</option>
                  <option :value="3">3</option>
                </select>
                <div class="game-description">
                  <label for="">
                    Match{{ game.matchesPlayed.length > 1 ? "s" : "" }}
                    {{ new Intl.ListFormat("fr-CA", { style: "long", type: "conjunction" }).format(game.matchesPlayed.map(String)) }}
                  </label>
                  <p>{{ game.description }}</p>
                </div>
              </div>
            </div>
            <div class="vertical-line"></div>
            <div class="question-selected">
              <div class="position-numbers">
                <h2 v-for="i in 3" :key="i">{{ i }}</h2>
              </div>
              <TransitionGroup name="music-box-list" tag="div" class="question-selected-wrapper" @before-leave="onBeforeLeave">
                <div
                  v-for="i in 3"
                  :key="'box-' + (gameStore.specialGames.find((game) => game.position == i)?.idGame ?? 'empty-' + i)"
                  class="selected-music-box"
                  :class="{ 'drag-over': isDragOver(gameStore.specialGames, i) }"
                  :draggable="!!gameStore.specialGames.find((game) => game.position == i)"
                  @dragstart="onDragStart(gameStore.specialGames, gameStore.specialGames.find((game) => game.position == i)?.idGame, 3, $event, true)"
                  @dragend="onDragEnd"
                  @dragover="onSlotDragOver(gameStore.specialGames, i, $event)"
                  @dragleave="onSlotDragLeave"
                  @drop="onSlotDrop(gameStore.specialGames, i, 3)"
                >
                  <MusicBox
                    :class="{ invisible: !gameStore.specialGames.find((game) => game.position == i) }"
                    :gameName="gameStore.specialGames.find((game) => game.position == i)?.gameName ?? ''"
                  ></MusicBox>
                </div>
              </TransitionGroup>
            </div>
          </div>
        </div>
      </div>
      <div>
        <button v-if="completed" @click="">Envoyer</button>
      </div>
    </div>
  </div>
</template>

<style scoped>
h4 {
  margin-top: -1.5rem;
}

h2 {
  margin-bottom: 0rem;
}

.question-header h2 {
  margin-bottom: 1.5rem;
}

.music-boxes-container-preview {
  width: 100%;
  display: flex;
  flex-direction: column;
  align-items: center;
}

/* .survey-header {
  margin-bottom: 2rem;
} */

.survey-content {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.survey-questions {
  width: 100%;
}

.question-header > label {
  text-align: center;
  width: 100%;
  margin-top: -1rem;
  margin-bottom: 1rem;
  display: block;
}

.question-elements {
  display: grid;
  grid-template-columns: 1fr 5px 200px;
  gap: 1rem;
  width: 100%;
}

.question-options {
  display: flex;
  flex-wrap: wrap;
  /* justify-content: space-between; */
  gap: 1rem 0.5rem;
  height: max-content;
}

.music-box-element {
  width: 200px;
  display: grid;
  grid-template-rows: 64px 0.25rem 1fr;
  align-items: flex-start;
  align-content: flex-start;
  justify-items: center;
  gap: 0.5rem;
}

.music-box-element select {
  width: 2.25rem;
  margin-top: -1rem;
  z-index: 2;
}

.music-box-element :deep(.music-box) {
  cursor: grab;
}

.music-box-element :deep(.music-box):active {
  cursor: grabbing;
}

.selected-music-box :deep(.music-box) {
  cursor: grab;
}

.selected-music-box :deep(.music-box):active {
  cursor: grabbing;
}

/* .music-box-element .music-box {
  padding: 4% 4%;
  width: calc(100% - 8%);
} */

.game-description {
  /* grid-column: 1/3; */
  margin: 0;
  margin-top: 0.5rem;
  text-align: center;
}

.game-description p {
  margin-top: 0.25rem;
}

.game-description label {
  font-style: italic;
  line-height: 1;
}

.line-image {
  width: 5px;
  aspect-ratio: 15/1920;
}

.vertical-line {
  width: 5px;
  height: 100%;
  background-image: url("../assets/images/vertical-line.jpg");
  background-size: cover;
  background-position: center;
}

.question-selected {
  display: grid;
  grid-template-columns: 1rem 1fr;
  gap: 0.5rem;
  width: 100%;
}

.position-numbers {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  align-items: center;
}

.position-numbers h2 {
  margin: 0;
  height: 64px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.question-selected-wrapper {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  width: 100%;
  position: relative;
}

.selected-music-box {
  width: 176.217px;
  height: 64px;
  box-sizing: border-box;
  display: flex;
  align-items: center;
  border: 2px dashed transparent;
  border-radius: 5px;
}

.selected-music-box.drag-over {
  border-color: goldenrod;
  background-color: rgba(218, 165, 32, 0.1);
}

/* Transitions pour TransitionGroup
.music-box-list-move {
  transition: transform 35s ease;
}

.music-box-list-enter-active,
.music-box-list-leave-active {
  transition: all 35s ease;
}

.music-box-list-enter-from {
  opacity: 0;
  transform: translateX(30px);
}

.music-box-list-leave-to {
  opacity: 0;
  transform: translateX(30px);
}

.music-box-list-leave-active {
  position: absolute;
}

.music-box-list-leave-active :deep(.music-box) {
  container-type: normal;
} */

@media (max-width: 500px) {
}
</style>
