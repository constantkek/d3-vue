<template>
  <d3-container />
</template>

<script lang="ts">
import { defineComponent } from "vue";
import D3Container from "./components/D3Container.vue";

export default defineComponent({
  name: "App",
  components: {
    D3Container,
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
