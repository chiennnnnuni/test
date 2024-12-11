<template>
  <div class="video-background">
    <video ref="video" autoplay loop muted>
      <source src="@/assets/bg.mp4" type="video/mp4">
    </video>
    <MusicPlayer @loaded="onMusicPlayerLoaded"/>
    <transition name="fade">
      <Loading v-if="isLoading"/>
    </transition>
  </div>
</template>

<script>
import MusicPlayer from '@/components/MusicPlayer.vue';
import Loading from '@/components/Loading.vue';

export default {
  components: {
    MusicPlayer,
    Loading,
  },
  data() {
    return {
      isLoading: true,
      videoLoaded: null,
      musicPlayerLoaded: null,
    };
  },
  methods: {
    onMusicPlayerLoaded() {
      this.musicPlayerLoaded = Promise.resolve();
    },
    async waitForAllLoaded() {
      try {
        await Promise.all([this.videoLoaded, this.musicPlayerLoaded]);
        setTimeout(() => {
          this.isLoading = false;
        }, 1000);
      } catch (err) {
        console.error('Error during loading');
      }
    },
  },
  mounted() {
    this.videoLoaded = new Promise((resolve) => {
      const video = this.$refs.video;
      if (video.readyState >= 3) {
        resolve();
      } else {
        video.addEventListener('canplay', resolve, { once: true });
      }
    });

    this.waitForAllLoaded();
  }
};
</script>