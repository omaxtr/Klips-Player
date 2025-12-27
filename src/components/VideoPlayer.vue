<template>
  <div class="video-container glass" @mousemove="showControls" @mouseleave="hideControls">
    <video 
      ref="videoRef"
      :src="clip.url"
      @timeupdate="updateProgress"
      @loadedmetadata="onLoaded"
      @click="togglePlay"
      class="main-video"
    ></video>

    <!-- Custom Controls overlay -->
    <div class="controls" :class="{ visible: controlsVisible }">
      <div class="progress-container" @click="seek">
        <div class="progress-bg">
          <div class="progress-bar" :style="{ width: progress + '%' }"></div>
        </div>
      </div>

      <div class="buttons">
        <div class="left">
          <button @click="togglePlay" class="control-btn pulse">
            <template v-if="isPlaying">
              <svg viewBox="0 0 24 24" width="24" height="24"><path fill="currentColor" d="M6 19h4V5H6v14zm8-14v14h4V5h-4z"/></svg>
            </template>
            <template v-else>
              <svg viewBox="0 0 24 24" width="24" height="24"><path fill="currentColor" d="M8 5v14l11-7z"/></svg>
            </template>
          </button>
          <span class="time">{{ formatTime(currentTime) }} / {{ formatTime(duration) }}</span>
        </div>

        <div class="center">
          <h3 class="video-title">{{ clip.title }}</h3>
        </div>

        <div class="right">
          <button @click="toggleMute" class="control-btn">
            <template v-if="isMuted">
              <svg viewBox="0 0 24 24" width="24" height="24"><path fill="currentColor" d="M16.5 12c0-1.77-1.02-3.29-2.5-4.03v8.05c1.48-.73 2.5-2.25 2.5-4.02zM14 3.23v2.06c2.89.86 5 3.54 5 6.71s-2.11 5.85-5 6.71v2.06c4.01-.91 7-4.49 7-8.77s-2.99-7.86-7-8.77zM3 9v6h4l5 5V4L7 9H3z"/></svg>
            </template>
            <template v-else>
              <svg viewBox="0 0 24 24" width="24" height="24"><path fill="currentColor" d="M3 9v6h4l5 5V4L7 9H3zm13.5 3c0-1.77-1.02-3.29-2.5-4.03v8.05c1.48-.73 2.5-2.25 2.5-4.02z"/></svg>
            </template>
          </button>
          <button @click="toggleFullscreen" class="control-btn">
            <svg viewBox="0 0 24 24" width="24" height="24"><path fill="currentColor" d="M7 14H5v5h5v-2H7v-3zm-2-4h2V7h3V5H5v5zm12 7h-3v2h5v-5h-2v3zM14 5v2h3v3h2V5h-5z"/></svg>
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  props: {
    clip: Object
  },
  data() {
    return {
      isPlaying: false,
      isMuted: false,
      progress: 0,
      currentTime: 0,
      duration: 0,
      controlsVisible: true,
      timer: null
    }
  },
  watch: {
    clip() {
      this.isPlaying = false;
      this.progress = 0;
      this.currentTime = 0;
      this.$nextTick(() => {
        if (this.$refs.videoRef) {
          this.$refs.videoRef.load();
        }
      });
    }
  },
  methods: {
    togglePlay() {
      const v = this.$refs.videoRef;
      if (v.paused) {
        v.play();
        this.isPlaying = true;
      } else {
        v.pause();
        this.isPlaying = false;
      }
    },
    toggleMute() {
      this.isMuted = !this.isMuted;
      this.$refs.videoRef.muted = this.isMuted;
    },
    updateProgress() {
      const v = this.$refs.videoRef;
      this.currentTime = v.currentTime;
      this.progress = (v.currentTime / v.duration) * 100;
      if (v.ended) this.isPlaying = false;
    },
    onLoaded() {
      this.duration = this.$refs.videoRef.duration;
    },
    seek(e) {
      const rect = e.currentTarget.getBoundingClientRect();
      const pos = (e.clientX - rect.left) / rect.width;
      this.$refs.videoRef.currentTime = pos * this.duration;
    },
    formatTime(seconds) {
      const min = Math.floor(seconds / 60);
      const sec = Math.floor(seconds % 60);
      return `${min}:${sec < 10 ? '0' : ''}${sec}`;
    },
    showControls() {
      this.controlsVisible = true;
      clearTimeout(this.timer);
      this.timer = setTimeout(() => {
        if (this.isPlaying) this.controlsVisible = false;
      }, 3000);
    },
    hideControls() {
      if (this.isPlaying) this.controlsVisible = false;
    },
    toggleFullscreen() {
      const v = this.$refs.videoRef;
      if (v.requestFullscreen) v.requestFullscreen();
      else if (v.webkitRequestFullscreen) v.webkitRequestFullscreen();
      else if (v.msRequestFullscreen) v.msRequestFullscreen();
    }
  },
  mounted() {
    this.showControls();
  }
}
</script>

<style scoped>
.video-container {
  position: relative;
  width: 100%;
  aspect-ratio: 16 / 9;
  border-radius: var(--radius);
  overflow: hidden;
  background: #000;
  box-shadow: 0 20px 50px rgba(0,0,0,0.5);
}

.main-video {
  width: 100%;
  height: 100%;
  object-fit: contain;
}

.controls {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  background: linear-gradient(transparent, rgba(0,0,0,0.8));
  padding: 20px;
  padding-top: 40px;
  display: flex;
  flex-direction: column;
  gap: 15px;
  opacity: 0;
  transition: opacity 0.3s ease;
  pointer-events: none;
}

.controls.visible {
  opacity: 1;
  pointer-events: all;
}

.progress-container {
  height: 4px;
  cursor: pointer;
  display: flex;
  align-items: center;
}

.progress-bg {
  width: 100%;
  height: 4px;
  background: rgba(255,255,255,0.2);
  border-radius: 2px;
  position: relative;
  transition: height 0.2s ease;
}

.progress-container:hover .progress-bg {
  height: 6px;
}

.progress-bar {
  height: 100%;
  background: var(--accent);
  border-radius: 2px;
  position: relative;
}

.progress-bar::after {
  content: '';
  position: absolute;
  right: -6px;
  top: 50%;
  transform: translateY(-50%);
  width: 12px;
  height: 12px;
  background: #fff;
  border-radius: 50%;
  opacity: 0;
  transition: opacity 0.2s ease;
}

.progress-container:hover .progress-bar::after {
  opacity: 1;
}

.buttons {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.left, .right {
  display: flex;
  align-items: center;
  gap: 15px;
}

.control-btn {
  background: none;
  border: none;
  color: #fff;
  cursor: pointer;
  padding: 5px;
  border-radius: 50%;
  transition: var(--transition);
  display: flex;
  align-items: center;
  justify-content: center;
}

.control-btn:hover {
  background: rgba(255,255,255,0.1);
  color: var(--accent-light);
}

.pulse:active {
  transform: scale(0.9);
}

.time {
  font-size: 14px;
  font-weight: 500;
  color: rgba(255,255,255,0.9);
}

.video-title {
  font-size: 16px;
  font-weight: 600;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 300px;
}
</style>
