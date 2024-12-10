<template>
  <div class="video-background">
    <video autoplay loop muted>
      <source src="./assets/bg.mp4" type="video/mp4">
    </video>
    <MusicPlayer @loaded="handleLoaded"/>
    <transition name="fade">
      <Loading v-if="isLoading"/>
    </transition>
  </div>
</template>

<script>
import MusicPlayer from './components/MusicPlayer.vue';
import Loading from './components/Loading.vue';

export default {
  components: {
    MusicPlayer,
    Loading,
  },
  data() {
    return {
      isLoading: true,
    };
  },
  methods: {
    handleLoaded() {
      this.isLoading = false;
    },
  },
};
</script>