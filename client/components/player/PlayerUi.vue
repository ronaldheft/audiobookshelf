<template>
  <div class="w-full -mt-6">
    <div class="w-full relative mb-1">
      <div class="absolute -top-10 md:top-0 right-0 lg:right-2 flex items-center h-full">
        <!-- <span class="material-icons text-2xl cursor-pointer" @click="toggleFullscreen(true)">expand_less</span> -->

        <controls-volume-control ref="volumeControl" v-model="volume" @input="setVolume" class="mx-2 hidden md:block" />

        <div class="cursor-pointer text-gray-300 hover:text-white mx-1 lg:mx-2" @mousedown.prevent @mouseup.prevent @click.stop="$emit('showSleepTimer')">
          <span v-if="!sleepTimerSet" class="material-icons text-2xl sm:text-2.5xl">snooze</span>
          <div v-else class="flex items-center">
            <span class="material-icons text-lg text-warning">snooze</span>
            <p class="text-xl text-warning font-mono font-semibold text-center px-0.5 pb-0.5" style="min-width: 30px">{{ sleepTimerRemainingString }}</p>
          </div>
        </div>

        <div v-if="!isPodcast" class="cursor-pointer text-gray-300 hover:text-white mx-1 lg:mx-2" @mousedown.prevent @mouseup.prevent @click.stop="$emit('showBookmarks')">
          <span class="material-icons text-2xl sm:text-2.5xl">{{ bookmarks.length ? 'bookmarks' : 'bookmark_border' }}</span>
        </div>

        <div v-if="chapters.length" class="cursor-pointer text-gray-300 hover:text-white mx-1 lg:mx-2" @mousedown.prevent @mouseup.prevent @click.stop="showChapters">
          <span class="material-icons text-2xl sm:text-3xl">format_list_bulleted</span>
        </div>

        <ui-tooltip v-if="chapters.length" direction="top" :text="useChapterTrack ? 'Use full track' : 'Use chapter track'">
          <div class="cursor-pointer text-gray-300 mx-1 lg:mx-2 hover:text-white" @mousedown.prevent @mouseup.prevent @click.stop="setUseChapterTrack">
            <span class="material-icons text-2xl sm:text-3xl transform transition-transform" :class="useChapterTrack ? 'rotate-180' : ''">timelapse</span>
          </div>
        </ui-tooltip>
      </div>

      <player-playback-controls :loading="loading" :seek-loading="seekLoading" :playback-rate.sync="playbackRate" :paused="paused" :has-next-chapter="hasNextChapter" @prevChapter="prevChapter" @nextChapter="nextChapter" @jumpForward="jumpForward" @jumpBackward="jumpBackward" @setPlaybackRate="setPlaybackRate" @playPause="playPause" />
    </div>

    <player-track-bar ref="trackbar" :loading="loading" :chapters="chapters" :duration="duration" :current-chapter="currentChapter" @seek="seek" />

    <div class="flex">
      <p ref="currentTimestamp" class="font-mono text-xxs sm:text-sm text-gray-100 pointer-events-auto">00:00:00</p>
      <p class="font-mono text-sm hidden sm:block text-gray-100 pointer-events-auto">&nbsp;/&nbsp;{{ progressPercent }}%</p>
      <div class="flex-grow" />
      <p class="text-xs sm:text-sm text-gray-300 pt-0.5">
        {{ currentChapterName }} <span v-if="useChapterTrack" class="text-xs text-gray-400">&nbsp;({{ currentChapterIndex + 1 }} of {{ chapters.length }})</span>
      </p>
      <div class="flex-grow" />
      <p class="font-mono text-xxs sm:text-sm text-gray-100 pointer-events-auto">{{ timeRemainingPretty }}</p>
    </div>

    <modals-chapters-modal v-model="showChaptersModal" :current-chapter="currentChapter" :chapters="chapters" @select="selectChapter" />
  </div>
</template>

<script>
export default {
  props: {
    loading: Boolean,
    paused: Boolean,
    chapters: {
      type: Array,
      default: () => []
    },
    bookmarks: {
      type: Array,
      default: () => []
    },
    sleepTimerSet: Boolean,
    sleepTimerRemaining: Number,
    isPodcast: Boolean
  },
  data() {
    return {
      volume: 1,
      playbackRate: 1,
      audioEl: null,
      seekLoading: false,
      showChaptersModal: false,
      currentTime: 0,
      duration: 0,
      useChapterTrack: false
    }
  },
  computed: {
    sleepTimerRemainingString() {
      var rounded = Math.round(this.sleepTimerRemaining)
      if (rounded < 90) {
        return `${rounded}s`
      }
      var minutesRounded = Math.round(rounded / 60)
      if (minutesRounded < 90) {
        return `${minutesRounded}m`
      }
      var hoursRounded = Math.round(minutesRounded / 60)
      return `${hoursRounded}h`
    },
    token() {
      return this.$store.getters['user/getToken']
    },
    timeRemaining() {
      if (this.useChapterTrack && this.currentChapter) {
        var currChapTime = this.currentTime - this.currentChapter.start
        return (this.currentChapterDuration - currChapTime) / this.playbackRate
      }
      return (this.duration - this.currentTime) / this.playbackRate
    },
    timeRemainingPretty() {
      if (this.timeRemaining < 0) {
        return this.$secondsToTimestamp(this.timeRemaining * -1)
      }
      return '-' + this.$secondsToTimestamp(this.timeRemaining)
    },
    progressPercent() {
      const duration = this.useChapterTrack ? this.currentChapterDuration : this.duration
      const time = this.useChapterTrack ? Math.max(this.currentTime - this.currentChapterStart) : this.currentTime

      if (!duration) return 0
      return Math.round((100 * time) / duration)
    },
    currentChapter() {
      return this.chapters.find((chapter) => chapter.start <= this.currentTime && this.currentTime < chapter.end)
    },
    currentChapterName() {
      return this.currentChapter ? this.currentChapter.title : ''
    },
    currentChapterDuration() {
      if (!this.currentChapter) return 0
      return this.currentChapter.end - this.currentChapter.start
    },
    currentChapterStart() {
      if (!this.currentChapter) return 0
      return this.currentChapter.start
    },
    isFullscreen() {
      return this.$store.state.playerIsFullscreen
    },
    currentChapterIndex() {
      if (!this.currentChapter) return 0
      return this.chapters.findIndex((ch) => ch.id === this.currentChapter.id)
    },
    hasNextChapter() {
      if (!this.chapters.length) return false
      return this.currentChapterIndex < this.chapters.length - 1
    }
  },
  methods: {
    toggleFullscreen(isFullscreen) {
      this.$store.commit('setPlayerIsFullscreen', isFullscreen)

      var videoPlayerEl = document.getElementById('video-player')
      if (videoPlayerEl) {
        if (isFullscreen) {
          videoPlayerEl.style.width = '100vw'
          videoPlayerEl.style.height = '100vh'
          videoPlayerEl.style.top = '0px'
          videoPlayerEl.style.left = '0px'
        } else {
          videoPlayerEl.style.width = '384px'
          videoPlayerEl.style.height = '216px'
          videoPlayerEl.style.top = 'unset'
          videoPlayerEl.style.bottom = '80px'
          videoPlayerEl.style.left = '16px'
        }
      }
    },
    setDuration(duration) {
      this.duration = duration
    },
    setCurrentTime(time) {
      this.currentTime = time
      this.updateTimestamp()
      if (this.$refs.trackbar) this.$refs.trackbar.setCurrentTime(time)
    },
    playPause() {
      this.$emit('playPause')
    },
    jumpBackward() {
      this.$emit('jumpBackward')
    },
    jumpForward() {
      this.$emit('jumpForward')
    },
    increaseVolume() {
      if (this.volume >= 1) return
      this.volume = Math.min(1, this.volume + 0.1)
      this.setVolume(this.volume)
    },
    decreaseVolume() {
      if (this.volume <= 0) return
      this.volume = Math.max(0, this.volume - 0.1)
      this.setVolume(this.volume)
    },
    setVolume(volume) {
      this.$emit('setVolume', volume)
    },
    toggleMute() {
      if (this.$refs.volumeControl && this.$refs.volumeControl.toggleMute) {
        this.$refs.volumeControl.toggleMute()
      }
    },
    increasePlaybackRate() {
      var rates = [0.25, 0.5, 0.8, 1, 1.3, 1.5, 2, 2.5, 3]
      var currentRateIndex = rates.findIndex((r) => r === this.playbackRate)
      if (currentRateIndex >= rates.length - 1) return
      this.playbackRate = rates[currentRateIndex + 1] || 1
      this.playbackRateChanged(this.playbackRate)
    },
    decreasePlaybackRate() {
      var rates = [0.25, 0.5, 0.8, 1, 1.3, 1.5, 2, 2.5, 3]
      var currentRateIndex = rates.findIndex((r) => r === this.playbackRate)
      if (currentRateIndex <= 0) return
      this.playbackRate = rates[currentRateIndex - 1] || 1
      this.playbackRateChanged(this.playbackRate)
    },
    setPlaybackRate(playbackRate) {
      this.$emit('setPlaybackRate', playbackRate)
    },
    selectChapter(chapter) {
      this.seek(chapter.start)
      this.showChaptersModal = false
    },
    setUseChapterTrack() {
      var useChapterTrack = !this.useChapterTrack
      this.useChapterTrack = useChapterTrack
      if (this.$refs.trackbar) this.$refs.trackbar.setUseChapterTrack(useChapterTrack)

      this.$store.dispatch('user/updateUserSettings', { useChapterTrack }).catch((err) => {
        console.error('Failed to update settings', err)
      })
      this.updateTimestamp()
    },
    checkUpdateChapterTrack() {
      // Changing media in player may not have chapters
      if (!this.chapters.length && this.useChapterTrack) {
        this.useChapterTrack = false
        if (this.$refs.trackbar) this.$refs.trackbar.setUseChapterTrack(this.useChapterTrack)
      }
    },
    seek(time) {
      this.$emit('seek', time)
    },
    restart() {
      this.seek(0)
    },
    prevChapter() {
      if (!this.currentChapter || this.currentChapterIndex === 0) {
        return this.restart()
      }
      var timeInCurrentChapter = this.currentTime - this.currentChapter.start
      if (timeInCurrentChapter <= 3 && this.chapters[this.currentChapterIndex - 1]) {
        var prevChapter = this.chapters[this.currentChapterIndex - 1]
        this.seek(prevChapter.start)
      } else {
        this.seek(this.currentChapter.start)
      }
    },
    nextChapter() {
      if (!this.currentChapter || !this.hasNextChapter) return
      var nextChapter = this.chapters[this.currentChapterIndex + 1]
      this.seek(nextChapter.start)
    },
    setStreamReady() {
      if (this.$refs.trackbar) this.$refs.trackbar.setPercentageReady(1)
    },
    setChunksReady(chunks, numSegments) {
      var largestSeg = 0
      for (let i = 0; i < chunks.length; i++) {
        var chunk = chunks[i]
        if (typeof chunk === 'string') {
          var chunkRange = chunk.split('-').map((c) => Number(c))
          if (chunkRange.length < 2) continue
          if (chunkRange[1] > largestSeg) largestSeg = chunkRange[1]
        } else if (chunk > largestSeg) {
          largestSeg = chunk
        }
      }
      var percentageReady = largestSeg / numSegments
      if (this.$refs.trackbar) this.$refs.trackbar.setPercentageReady(percentageReady)
    },
    updateTimestamp() {
      var ts = this.$refs.currentTimestamp
      if (!ts) {
        console.error('No timestamp el')
        return
      }
      const time = this.useChapterTrack ? Math.max(0, this.currentTime - this.currentChapterStart) : this.currentTime
      var currTimeClean = this.$secondsToTimestamp(time)
      ts.innerText = currTimeClean
    },
    setBufferTime(bufferTime) {
      if (this.$refs.trackbar) this.$refs.trackbar.setBufferTime(bufferTime)
    },
    showChapters() {
      if (!this.chapters.length) return
      this.showChaptersModal = !this.showChaptersModal
    },
    init() {
      this.playbackRate = this.$store.getters['user/getUserSetting']('playbackRate') || 1

      var _useChapterTrack = this.$store.getters['user/getUserSetting']('useChapterTrack') || false
      this.useChapterTrack = this.chapters.length ? _useChapterTrack : false

      if (this.$refs.trackbar) this.$refs.trackbar.setUseChapterTrack(this.useChapterTrack)
      this.$emit('setPlaybackRate', this.playbackRate)
    },
    settingsUpdated(settings) {
      if (settings.playbackRate && this.playbackRate !== settings.playbackRate) {
        this.setPlaybackRate(settings.playbackRate)
      }
    },
    closePlayer() {
      if (this.isFullscreen) {
        this.toggleFullscreen(false)
        return
      }

      if (this.loading) return
      this.$emit('close')
    },
    hotkey(action) {
      if (action === this.$hotkeys.AudioPlayer.PLAY_PAUSE) this.playPause()
      else if (action === this.$hotkeys.AudioPlayer.JUMP_FORWARD) this.jumpForward()
      else if (action === this.$hotkeys.AudioPlayer.JUMP_BACKWARD) this.jumpBackward()
      else if (action === this.$hotkeys.AudioPlayer.VOLUME_UP) this.increaseVolume()
      else if (action === this.$hotkeys.AudioPlayer.VOLUME_DOWN) this.decreaseVolume()
      else if (action === this.$hotkeys.AudioPlayer.MUTE_UNMUTE) this.toggleMute()
      else if (action === this.$hotkeys.AudioPlayer.SHOW_CHAPTERS) this.showChapters()
      else if (action === this.$hotkeys.AudioPlayer.INCREASE_PLAYBACK_RATE) this.increasePlaybackRate()
      else if (action === this.$hotkeys.AudioPlayer.DECREASE_PLAYBACK_RATE) this.decreasePlaybackRate()
      else if (action === this.$hotkeys.AudioPlayer.CLOSE) this.closePlayer()
    }
  },
  mounted() {
    this.$store.commit('user/addSettingsListener', { id: 'audioplayer', meth: this.settingsUpdated })
    this.init()
    this.$eventBus.$on('player-hotkey', this.hotkey)
  },
  beforeDestroy() {
    this.$store.commit('user/removeSettingsListener', 'audioplayer')
    this.$eventBus.$off('player-hotkey', this.hotkey)
  }
}
</script>

<style>
.loadingTrack {
  animation-name: loadingTrack;
  animation-duration: 1s;
  animation-iteration-count: infinite;
}
@keyframes loadingTrack {
  0% {
    left: -25%;
  }
  100% {
    left: 100%;
  }
}
</style>