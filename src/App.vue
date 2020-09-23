<template>
  <div id="app">
    <threeWorld></threeWorld>
    <!-- <threeExamples></threeExamples> -->
  </div>
</template>

<script>
import threeWorld from "./components/threeWorld.vue";
// import threeExamples from "./components/threeExamples.vue";

export default {
  name: "App",
  components: {
    threeWorld,
    // threeExamples,
  },
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  height: 100vh;
  width: 100vw;
  margin: 0;
  overflow: hidden;
}
</style>
