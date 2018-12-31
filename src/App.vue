<template>
  <div id="app">
    <!-- <mnist /> -->
    <!-- <polynome/> -->
    <sound/>
  </div>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator';
import Mnist from './components/mnist.vue';
import Polynome from './components/polynome.vue';
import Sound from './components/sound.vue';

@Component({
  components: {
    Mnist,
    Polynome,
    Sound,
  },
})
export default class App extends Vue {}
</script>

<style lang="scss">
body {
    height: 100%;
    width: 100%;
}

#app {
  position: absolute;
  top: 0;
  right: 0;
  bottom: 0;
  left: 0;
}
</style>
