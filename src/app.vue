<template>
  <div id="app">

    <div class="bg-light text-right p-4 mb-4">
      <img src="./assets/logo-deluge.png">
    </div>

    <div class="container">
      <deluge-song-merger></deluge-song-merger>
    </div>

    <div class="bg-light text-right p-4 mt-4">
      <div class="small text-muted text-center">
        <div>
          From Berlin with ♥
        </div>
        <div class="small">
          Copyright (c) 2019 Oliver Leics &lt;oliver.leics@gmail.com&gt;
        </div>
      </div>
    </div>

  </div>
</template>

<script>
import DelugeSongMerger from './components/deluge-song-merger.vue'

export default {
  name: 'app',
  components: {
    'deluge-song-merger': DelugeSongMerger
  }
}
</script>

<style>
#app {
  /* font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px; */
}
</style>
