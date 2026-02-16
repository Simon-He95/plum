<script setup lang="ts">
import Plum from './plum.vue'
import { isDark, toggleDark } from './toggleDark'
</script>

<template>
  <nav class="nav">
    <div class="title-wrap">
      <p class="kicker">Generative Collection</p>
      <h1 class="title">Plum Study No. 1</h1>
    </div>
    <div class="right">
      <a href="https://github.com/Simon-He95/plum" target="_blank" rel="noreferrer" title="GitHub">
        <span i-uil:github-alt></span>
      </a>
      <button type="button" class="toggle select-none" title="Toggle Color Scheme" @click="toggleDark">
        <span v-show="isDark" i-ri:moon-line></span>
        <span v-show="!isDark" i-ri:sun-line></span>
      </button>
    </div>
  </nav>
  <Plum />
</template>

<style scoped>
.nav {
  padding: 1.4rem 1.2rem 0.9rem;
  width: 100%;
  display: grid;
  grid-template-columns: auto max-content;
  align-items: center;
  gap: 16px;
  box-sizing: border-box;
  max-width: 1040px;
  margin: 0 auto;
}

.title-wrap {
  display: grid;
  gap: 3px;
  text-align: left;
}

.kicker {
  margin: 0;
  font-size: 0.68rem;
  text-transform: uppercase;
  letter-spacing: 0.14em;
  color: var(--muted);
}

.title {
  margin: 0;
  font-size: 1.4rem;
  line-height: 1.05;
  letter-spacing: 0.015em;
  font-family: 'Cormorant Garamond', 'Noto Serif SC', 'Source Han Serif SC', serif;
  font-weight: 600;
}

.nav a,
.toggle {
  cursor: pointer;
  text-decoration: none;
  color: inherit;
  transition: opacity 0.2s ease, transform 0.2s ease;
  opacity: 0.72;
  outline: none;
  border: 1px solid rgba(114, 106, 91, 0.35);
  background: transparent;
  width: 36px;
  height: 36px;
  border-radius: 999px;
  font-size: 1.15rem;
  line-height: 1;
  display: grid;
  place-items: center;
}

.nav a:hover,
.toggle:hover {
  opacity: 1;
  transform: translateY(-1px);
}

.nav .right {
  display: grid;
  grid-gap: 0.65rem;
  grid-auto-flow: column;
}

.nav .right > * {
  margin: auto;
}

@media (max-width: 680px) {
  .nav {
    padding-top: 1rem;
    grid-template-columns: 1fr;
  }

  .title-wrap {
    text-align: center;
  }
}
</style>
