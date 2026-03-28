<script setup lang="ts">
import SurveyMatch5 from "./components/SurveyMatch5.vue";
</script>

<template>
  <div id="app-content">
    <header>
      <img class="horizontal-logo" id="header-logo" src="./assets/images/Logo-SMALL.png" alt="Logo" />
      <img class="vertical-logo" id="header-logo" src="./assets/images/Logo-SMALL-VERT.png" alt="Logo" />
    </header>
    <SurveyMatch5 />
  </div>
</template>

<style>
.vertical-logo {
  display: none;
}

@media (max-width: 700px) {
  .horizontal-logo {
    display: none;
  }
  .vertical-logo {
    display: block;
  }

  header {
    display: flex;
    justify-content: center;
    align-items: center;
  }

  #header-logo {
    max-height: 120px;
  }
}
</style>
