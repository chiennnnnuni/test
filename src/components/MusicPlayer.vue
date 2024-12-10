<template>
  <div class="wrapper" id="music">
    <div class="title">《<span class="title-spacing">今日的</span>BGM》</div>
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
      nextCount: 0,
      priorityId: 0,
      prevTracks: [],
      currentTrack: {
        cover: ' '
      },
      duration: '--:--',
      currentTime: '--:--',
      barWidthPercent: 0,
      isPlaying: false,
      shuffleMode: false,
      preTrackBtnStyle: 0.3,
      liked: false,
    };
  },
  emits: ['loaded'],
  components: {
    SvgDefs,
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
      console.log(this.currentTrack);

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
      const oneLimitListened = limitedTracksId.some(id => this.prevTracks.includes(id));

      const selectNextTrack = (arr) => {
        const [opt1, opt2] = this.shuffleArray(arr);
        if (opt1.id === this.prevTracks[this.prevTracks.length - 1]) {
          return opt2;
        } else {
          return opt1;
        }
      };

      if (!oneLimitListened && this.priorityId) {
        this.currentTrack = this.findTrack(this.limitedPool, this.priorityId);
      } else if (this.limitedPool.length && this.nextCount < guaranteed && !oneLimitListened) {
        this.currentTrack = selectNextTrack(limitedPoolCopy);
      } else if (this.limitedPool.length && this.nextCount === guaranteed && !oneLimitListened) {
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

      if (withValidLimit.length || todayPriority) {
        const fillerTracks = shuffledNormalTracks.splice(0, 19);
        this.limitedPool = this.shuffleArray([...withValidLimit, ...fillerTracks]);
        this.tracksOfToday = [...this.limitedPool, ...shuffledNormalTracks];
      } else {
        this.tracksOfToday = shuffledNormalTracks;
      }

      if (todayPriority) {
        this.priorityId = todayPriority.id;

        const idx = Math.floor(Math.random() * 2); // 0 o 1
        this.limitedPool.splice(idx, 0, todayPriority);
        this.tracksOfToday.splice(idx, 0, todayPriority);
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
      this.audio.src = '';
      this.barWidthPercent = 0;
      console.log('pause');
      
    },
    setAudioSrc() {
      console.log('set audio src');
      console.log(this.isPlaying);
      
      this.audio = this.preloadedAudios.find(audio => {
        console.log(audio.currentSrc);
        // console.log(this.currentTrack.source);
        // console.log(audio.src === this.currentTrack.source);
        
        return audio.currentSrc === this.currentTrack.source
      });
      console.log(this.currentTrack);
      console.log(this.preloadedAudios);
      
      console.dir(this.audio);
      
      this.audio.ontimeupdate = () => {
        this.generateTime();
      };
      this.audio.onloadedmetadata = () => {
        console.log('on load meta data');
        
        this.duration = this.formatTime(this.audio.duration);
      };
      this.audio.onended = () => {
        console.log('on ended');
        
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
    }
  },
  // created() {
  //   this.audio = new Audio();
  //   this.audio.ontimeupdate = () => {
  //     this.generateTime();
  //   };
  //   this.audio.onloadedmetadata = () => {
  //     this.duration = this.formatTime(this.audio.duration);
  //   };
  //   this.audio.onended = () => {
  //     this.isPlaying = true;
  //     this.shuffleMode ? this.nextRandomTrack() : this.setAudioSrc();
  //   };
  // },
  mounted() {
    document.addEventListener('visibilitychange', this.handleVisibilityChange);
    this.limitedAndPriority();

    this.preloadedAudios = this.tracks.map(track => this.preloadAudio(track.source));

    this.currentTrack = this.tracksOfToday[0];
    this.setAudioSrc();

    setTimeout(() => {
      this.$emit('loaded');
    }, 1000); 
  },
  beforeDestroy() {
    document.removeEventListener('visibilitychange', this.handleVisibilityChange);
  },
}
</script>
