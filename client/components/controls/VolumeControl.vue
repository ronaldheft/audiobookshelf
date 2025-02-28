<template>
  <div class="relative" v-click-outside="clickOutside" @mouseover="mouseover" @mouseleave="mouseleave">
    <div class="cursor-pointer text-gray-300 hover:text-white" @mousedown.prevent @mouseup.prevent @click="clickVolumeIcon">
      <span class="material-icons text-2xl sm:text-3xl">{{ volumeIcon }}</span>
    </div>
    <transition name="menux">
      <div v-show="isOpen" class="volumeMenu h-6 absolute bottom-2 w-28 px-2 bg-bg shadow-sm rounded-lg" style="left: -116px">
        <div ref="volumeTrack" class="h-1 w-full bg-gray-500 my-2.5 relative cursor-pointer rounded-full" @mousedown="mousedownTrack" @click="clickVolumeTrack">
          <div class="bg-gray-100 h-full absolute left-0 top-0 pointer-events-none rounded-full" :style="{ width: volume * trackWidth + 'px' }" />
          <div class="w-2.5 h-2.5 bg-white shadow-sm rounded-full absolute pointer-events-none" :class="isDragging ? 'transform scale-125 origin-center' : ''" :style="{ left: cursorLeft + 'px', top: '-3px' }" />
        </div>
      </div>
    </transition>
  </div>
</template>

<script>
export default {
  props: {
    value: Number
  },
  data() {
    return {
      isOpen: false,
      isDragging: false,
      isHovering: false,
      posX: 0,
      lastValue: 0.5,
      isMute: false,
      trackWidth: 112 - 20,
      openTimeout: null
    }
  },
  computed: {
    volume: {
      get() {
        return this.value
      },
      set(val) {
        this.$emit('input', val)
      }
    },
    cursorLeft() {
      var left = this.trackWidth * this.volume
      return left - 3
    },
    volumeIcon() {
      if (this.volume <= 0) return 'volume_mute'
      else if (this.volume <= 0.5) return 'volume_down'
      else return 'volume_up'
    }
  },
  methods: {
    scroll(e) {
      if (!e || !e.wheelDeltaY) return
      if (e.wheelDeltaY > 0) {
        this.volume = Math.min(1, this.volume + 0.1)
      } else {
        this.volume = Math.max(0, this.volume - 0.1)
      }
    },
    mouseover() {
      if (!this.isHovering) {
        window.addEventListener('mousewheel', this.scroll)
      }
      this.isHovering = true
      this.setOpen()
    },
    mouseleave() {
      if (this.isHovering) {
        window.removeEventListener('mousewheel', this.scroll)
      }
      this.isHovering = false
    },
    setOpen() {
      this.isOpen = true
      clearTimeout(this.openTimeout)
      this.openTimeout = setTimeout(() => {
        if (!this.isHovering && !this.isDragging) {
          this.isOpen = false
        } else {
          this.setOpen()
        }
      }, 600)
    },
    mousemove(e) {
      var diff = this.posX - e.x
      this.posX = e.x
      var volShift = 0
      if (diff < 0) {
        // Volume up
        volShift = diff / this.trackWidth
      } else {
        // volume down
        volShift = diff / this.trackWidth
      }
      var newVol = this.volume - volShift
      newVol = Math.min(Math.max(0, newVol), 1)
      this.volume = newVol
      e.preventDefault()
    },
    mouseup(e) {
      if (this.isDragging) {
        this.isDragging = false
        document.body.removeEventListener('mousemove', this.mousemove)
        document.body.removeEventListener('mouseup', this.mouseup)
      }
    },
    mousedownTrack(e) {
      this.isDragging = true
      this.posX = e.x
      var vol = e.offsetX / this.trackWidth
      vol = Math.min(Math.max(vol, 0), 1)
      this.volume = vol
      document.body.addEventListener('mousemove', this.mousemove)
      document.body.addEventListener('mouseup', this.mouseup)
      e.preventDefault()
    },
    clickOutside() {
      this.isOpen = false
    },
    clickVolumeIcon() {
      this.isMute = !this.isMute
      if (this.isMute) {
        this.lastValue = this.volume
        this.volume = 0
      } else {
        this.volume = this.lastValue || 0.5
      }
    },
    toggleMute() {
      this.clickVolumeIcon()
    },
    clickVolumeTrack(e) {
      var vol = e.offsetX / this.trackWidth
      vol = Math.min(Math.max(vol, 0), 1)
      this.volume = vol
    }
  },
  mounted() {
    if (this.value === 0) {
      this.isMute = true
    }
  },
  beforeDestroy() {
    window.removeEventListener('mousewheel', this.scroll)
    document.body.removeEventListener('mousemove', this.mousemove)
    document.body.removeEventListener('mouseup', this.mouseup)
  }
}
</script>