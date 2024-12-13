<template>
  <div class="wrapper" id="music">
    <svg class="title">
      <use xlink:href="#font-title"></use>
    </svg>
    <div class="player">
      <div class="player__top">
        <!-- 圖片 -->
        <div class="player-cover">
          <transition name="move" mode="out-in">
            <div class="player-cover__item" :key="currentTrack.id" :style="{ backgroundImage: `url(${currentTrack.cover})`}">
              <img :src="currentTrack.cover"/>
            </div>
          </transition>
        </div>
        <!-- 控制按鈕 -->
        <div class="player-controls">
          <div class="player-controls__item" :class="{ active : liked }" @click="liked = !liked">
            <svg class="icon">
              <use xlink:href="#icon-heart-o"></use>
            </svg>
          </div>
          <div class="player-controls__item active" @click="shuffleMode = !shuffleMode">
            <svg class="icon">
              <use xlink:href="#icon-shuffle" v-if="shuffleMode"></use>
              <use xlink:href="#icon-repeat-1" v-else></use>
            </svg>
          </div>
          <div class="player-controls__item" :style="{'opacity': prevTracks.length === 0 ? 0.3 : 1 }" @click="playPrevTrack">
            <svg class="icon">
              <use xlink:href="#icon-prev"></use>
            </svg>
          </div>
          <div class="player-controls__item" @click="nextRandomTrack">
            <svg class="icon">
              <use xlink:href="#icon-next"></use>
            </svg>
          </div>
          <div class="player-controls__item -xl" @click="togglePlay">
            <svg class="icon">
              <use xlink:href="#icon-pause" v-if="isPlaying"></use>
              <use xlink:href="#icon-play" v-else></use>
            </svg>
          </div>
        </div>
      </div>
      <!-- 曲目資訊 -->
      <transition name="fade" mode="out-in">
        <div class="album-info" :key="currentTrack.id">
          <div class="album-info__title">
            <span v-if="currentTrack.limited" class="badge">
              {{ isNaN(currentTrack.limited) ? currentTrack.limited : currentTrack.limited+' '}}限定</span
              >{{ currentTrack.title || '--'}}
          </div>
          <div class="album-info__subTitle">{{ currentTrack.subTitle || '--' }}
            <a v-if="currentTrack.link" :href="currentTrack.link" target="_blank" style="color: inherit;">
              <svg class="icon" style="margin-bottom: -2px;">
                <use xlink:href="#icon-link"></use>
              </svg>
            </a>
          </div>
        </div>
      </transition>
      <!-- 進度條 -->
      <div class="progress" ref="progress">
        <div class="progress__duration">{{ duration }}</div>
        <div class="progress__bar" @click="clickProgress">
          <div class="progress__current active" :style="{ width : barWidthPercent }"></div>
        </div>
        <div class="progress__time">{{ currentTime }}</div>
      </div>
    </div>
    <div class="footer">
      <div>Inspired by
        <a href="https://github.com/itanand/mini-music-player" target="_blank" class="footer-link">itanand</a>.
      </div>
      <div>Background from
        <a href="https://www.vecteezy.com/video/2017224-small-particles-and-stars-moving-on-galaxy" target="_blank" class="footer-link">Vecteezy</a>.
      </div>
    </div>
  </div>
  <SvgDefs />
</template>

<script>
import { tracks } from '@/data/trackData.js';
import SvgDefs from '@/components/SvgDefs.vue';

export default {
  data() {
    return {
      audio: null,
      tracks,
      preloadedAudios: [],
      tracksOfToday: [],
      limitedPool: [],
      priorityId: 0,
      nextCount: 0,
      oneLimitListened: false,
      prevTracks: [],
      currentTrack: {
        cover: ' '
      },
      duration: '--:--',
      currentTime: '--:--',
      barWidthPercent: 0,
      isPlaying: false,
      shuffleMode: false,
      liked: false,
      imageCache: {},
    };
  },
  emits: ['loaded'],
  components: {
    SvgDefs,
  },
  watch: {
    "currentTrack.limited"(){
      if(this.currentTrack.limited){
        this.oneLimitListened = true;
      }
    }
  },
  methods: {
    togglePlay() {
      this.audio.paused ? this.audio.play() : this.audio.pause();
      this.isPlaying = !this.audio.paused;
    },
    generateTime() {
      // update progress bar
      let width = ((this.audio.currentTime / this.audio.duration) * 100).toFixed(2);
      this.barWidthPercent = width + "%";
      // update current time
      if(!this.audio.currentTime && !this.isPlaying){
        this.currentTime = '--:--';
      } else {
        this.currentTime = this.formatTime(this.audio.currentTime);
      }
    },
    formatTime(time) {
      const min = Math.floor(time / 60);
      const sec = Math.floor(time % 60);
      return `${min < 10 ? '0' : ''}${min}:${sec < 10 ? '0' : ''}${sec}`;
    },
    clickProgress(e) {
      this.isPlaying = true;

      const progressBar = this.$refs.progress;
      const progressBarRect = progressBar.getBoundingClientRect();

      let clickX = e.pageX - progressBarRect.left; 
      let ratio = this.updateBarWidth(clickX);

      this.audio.currentTime = this.audio.duration * ratio;
      this.audio.play();
    },
    updateBarWidth(clickX) {
      let progress = this.$refs.progress;
      let progressBarWidth = progress.querySelector('.progress__bar').offsetWidth;
      let ratio = clickX / progressBarWidth;

      this.barWidthPercent = (ratio * 100).toFixed(2) + "%";
      return ratio;
    },
    playPrevTrack() {
      if (!this.prevTracks.length){
        return
      }
      this.pauseAndReset();
      const prevTrackId = this.prevTracks[this.prevTracks.length - 1];
      this.currentTrack = this.findTrack(this.tracksOfToday, prevTrackId);
      this.prevTracks.pop();
      this.setAudioSrc();
    },
    nextRandomTrack(){
      this.pauseAndReset();
      this.prevTracks.push(this.currentTrack.id);
      this.shuffleMode = true;
      this.nextCount++;

      const guaranteed = 4;
      const limitedPoolCopy = JSON.parse(JSON.stringify(this.limitedPool));
      const limitedTracksId = this.limitedPool.filter(t => t.limited).map(obj => obj.id);

      const selectNextTrack = (arr) => {
        const [opt1, opt2] = this.shuffleArray(arr);
        if (opt1.id === this.prevTracks[this.prevTracks.length - 1]) {
          return opt2;
        } else {
          return opt1;
        }
      };

      if (!this.oneLimitListened && this.priorityId) {
        this.currentTrack = this.findTrack(this.limitedPool, this.priorityId);
      } else if (this.limitedPool.length && this.nextCount < guaranteed && !this.oneLimitListened) {
        this.currentTrack = selectNextTrack(limitedPoolCopy);
      } else if (this.limitedPool.length && this.nextCount === guaranteed && !this.oneLimitListened) {
        const randomIdx = Math.floor(Math.random() * limitedTracksId.length);
        this.currentTrack = this.findTrack(this.limitedPool, limitedTracksId[randomIdx]);
      } else {
        this.currentTrack = selectNextTrack(this.tracksOfToday);
      }

      this.setAudioSrc();
    },
    findTrack(arr, id) {
      return arr.find((t) => t.id === id );
    },
    getFactors(){
      const time = new Date();
      const month = time.getMonth() + 1;
      const date = time.getDate();
      const day = time.getDay();  // Sun: 0, Mon: 1 
      const mmdd = `${month < 10 ? '0' : ''}${month}${date < 10 ? '0' : ''}${date}`;
      const taiwanHour = time.toLocaleString('en-US', {
        timeZone: 'Asia/Taipei',
        hour: '2-digit',
        hour12: false
      })

      const result = [ mmdd ];

      if(day === 1){
        result.push('週一');
      }

      if(taiwanHour >= 21 || taiwanHour <= 3){
        result.push('夜間');
      }

      return result;
    },
    limitedAndPriority() {
      const validFactors = this.getFactors();
      const [todayPriority, withValidLimit, withoutLimit] = this.tracks.reduce(
        ([todayPriority, withValidLimit, withoutLimit], t) => {
          if (t.limited === validFactors[0]) {
            todayPriority = t
          } else if (t.limited && validFactors.includes(t.limited)) {
            withValidLimit.push(t);
          } else if (!t.limited) {
            withoutLimit.push(t);
          }
          return [todayPriority, withValidLimit, withoutLimit];
      },[null, [], []]);
      
      const shuffledNormalTracks = this.shuffleArray(withoutLimit);

      if (todayPriority || withValidLimit.length) {
        let priorityPool = [];
        if (todayPriority) {
          this.priorityId = todayPriority.id;
          const priorityFiller = shuffledNormalTracks.splice(0, 1)[0];
          const idx = Math.floor(Math.random() * 2); // 0 or 1
          priorityPool = idx === 0 ? [todayPriority, priorityFiller] : [priorityFiller, todayPriority];
        }

        const fillerTracks = shuffledNormalTracks.splice(0, 19);
        this.limitedPool = this.shuffleArray([...withValidLimit, ...fillerTracks]);

        this.limitedPool = [...priorityPool, ...this.limitedPool];
        this.tracksOfToday = [...this.limitedPool, ...shuffledNormalTracks];
      } else {
        this.tracksOfToday = shuffledNormalTracks;
      }
    },
    shuffleArray(array) {
      for (let i = array.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [array[i], array[j]] = [array[j], array[i]];
      }
      return array;
    },
    pauseAndReset() {
      this.audio.pause();
      this.audio.currentTime = 0;
      this.barWidthPercent = 0;
    },
    setAudioSrc() {
      this.audio = this.preloadedAudios.find(audio => audio.src === this.currentTrack.source);
      
      this.audio.ontimeupdate = () => {
        this.generateTime();
      };
      
      if (this.audio.readyState >= 1) { // HAVE_METADATA
        this.duration = this.formatTime(this.audio.duration);
      } else {
        this.audio.onloadedmetadata = () => {
            this.duration = this.formatTime(this.audio.duration);
        };
      }
      this.audio.onended = () => {
        this.isPlaying = true;
        this.shuffleMode ? this.nextRandomTrack() : this.audio.play();
      };

      this.isPlaying ? this.audio.play() : this.audio.pause();
    },
    handleVisibilityChange() {
      if (document.hidden) {
        this.audio.pause();
        this.isPlaying = false;
      }
    },
    preloadAudio(trackSource) {
      const audio = new Audio(trackSource);
      audio.preload = 'auto';
      return audio;
    },
    preloadImages(imageUrls) {
      imageUrls.forEach((url) => {
        const img = new Image();
        img.src = url;
        this.imageCache[url] = img;
      });
    },
  },
  mounted() {
    document.addEventListener('visibilitychange', this.handleVisibilityChange, 300);
    
    this.limitedAndPriority();

    this.preloadedAudios = this.tracksOfToday.map(t => this.preloadAudio(t.source));
    this.currentTrack = this.tracksOfToday[0];
    this.setAudioSrc();

    const coverUrls = this.tracksOfToday.map((t) => t.cover);
    this.preloadImages(coverUrls);
    
    this.$emit('loaded');
  },
  beforeDestroy() {
    document.removeEventListener('visibilitychange', this.handleVisibilityChange);
  },
}
</script>
