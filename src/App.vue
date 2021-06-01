<template>
  <div id="app" class="mx-auto w-4/5">
    <network
    ></network>
  </div>
</template>

<script>
import Network from "./components/NetworkC.vue";
export default {
  name: "app",
  components: {
    Network
  },
  data() {
    return {
      publicPath: process.env.BASE_URL,
    }
  },
};
</script>

<style>
#app {
  font-family: 'Akkurat', sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: #2c3e50;
}

@font-face {
  font-family: Gordita;
  src: url('./assets/fonts/Gordita.otf');
}

@font-face {
  font-family: Gordita;
  src: url('./assets/fonts/Gordita-Bold.otf');
  font-weight: bold;
}

@font-face {
  font-family: Gordita;
  src: url('./assets/fonts/Gordita-Regular-Italic.otf');
  font-style: italic;
}

@font-face {
  font-family: Akkurat;
  src: url('./assets/fonts/AkkuratLLWeb-Regular.woff');
}

@font-face {
  font-family: Akkurat;
  src: url('./assets/fonts/AkkuratLLWeb-Bold.woff');
  font-weight: bold;
}

@font-face {
  font-family: Akkurat;
  src: url('./assets/fonts/AkkuratLLWeb-Italic.woff');
  font-style: italic;
}

</style>

