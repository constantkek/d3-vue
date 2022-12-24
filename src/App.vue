<template>
  <div>
    <!-- <d3-container /> -->
    <d3-statistics />
  </div>
</template>

<script lang="ts">
import { defineComponent } from "vue";
// import D3Container from "./components/D3Container.vue";
import D3Statistics from "./components/D3Statistics.vue";

export default defineComponent({
  name: "App",
  components: {
    // D3Container,
    D3Statistics,
  },
});
</script>

<style lang="scss">
#app {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}
</style>
