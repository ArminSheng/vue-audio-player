<template>
  <div>
    <div
      :style="{backgroundImage: 'url(' + audio.pic + ')'}"
      class="audio-player">
      <div
        flex="main:center cross:center dir:top"
        class="player-mask"
        v-if="showMask">
        <div @click="switchVol(audio)" class="player-error-msg">
          {{ errMsg }}
        </div>
      </div>

      <div
        class="player-mask"
        v-if="loading && !showMask"
        flex="main:center cross:center dir:top">
        <div class="player-error-msg">
          <!-- {{ loading ? '加载中...' : '' }} -->
          <span class="mint-indicator-spin">
            <div class="mint-spinner-snake"
              style="border-top-color: rgb(255, 255, 255);  border-left-color: rgb(255, 255, 255); border-bottom-color: rgb(255, 255, 255); height: 32px; width: 32px;"></div>
          </span>
        </div>
      </div>

      <div class="audio-controller" flex="cross:center">
        <img
          @click="toggle"
          :src="playing ? icon_stop : icon_play"
          class="audio-controller-toggle">
        <span class="audio-time">{{ currentTime }}</span>
        <div
          @click="onClickProgress($event)"
          class="audio-progress"
          ref="progressBar"
          flex
          flex-box="1">
          <div
            :style="{width: progress + '%'}"
            :class="{'ease': needTransit}"
            ref="playedBar"
            class="audio-progress-bar audio-progress-bar-played">
            <div
              @click.prevent.stop
              @touchmove="onMoveIndicator"
              @touchstart="onMoveStartIndicator"
              @touchend="onMoveEndIndicator"
              class="audio-indicator">
              <div class="audio-indicator-inner"></div>
            </div>
          </div>

          <div
            :style="{width: buffered + '%'}"
            class="audio-progress-bar audio-progress-bar-buffered">
          </div>
        </div>
        <span class="audio-time">{{ endTime }}</span>
      </div>
    </div>
  </div>
</template>

<script>
  import icon_play from '../../assets/img/icon/btn_play_white.png'
  import icon_stop from '../../assets/img/icon/btn_stop_white.png'

  let startX = 0
  let startWidth = 0
  const UTILS = {
    convertSeconds (sec) {
      if (isNaN(sec)) {
        return '00:00'
      }

      const hour = Math.floor(sec / 3600)
      const minute = Math.floor((sec % 3600) / 60)
      const second = Math.floor(sec - hour * 3600 - minute * 60)
      return `${hour ? UTILS.cover(hour) + ':' : ''}` +
        `${UTILS.cover(minute)}:` +
        `${UTILS.cover(second)}`
    },

    cover (num) {
      return num < 10 ? '0' + num : num
    }
  }

  export default {
    name: 'audio-player',
    props: {
      audio: {
        type: Object,
        default () {
          return {
            url: '',
            pic: ''
          }
        }
      }
    },

    data () {
      return {
        icon_play,
        icon_stop,
        instance: new Audio(),
        playing: false,
        currentTime: '00:00',
        endTime: '00:00',
        progress: 0,
        showMask: false,
        errMsg: '',
        loading: false,
        buffered: 0,
        needTransit: true
      }
    },

    watch: {
      audio (val) {
        this.switchVol(val)
      }
    },

    mounted () {
      this.init()
    },

    beforeDestroy () {
      this.destroyInstance()
    },

    methods: {
      init () {
        const {
          audio,
          instance,
          ontimeupdate,
          onerror,
          oncanplay,
          onplay,
          onpause,
          onwaiting,
          onprogress,
          onended
        } = this

        instance.src = audio.url

        // Binding event
        // instance.preload = true
        instance.oncanplay = oncanplay
        instance.ontimeupdate = ontimeupdate
        instance.onerror = onerror
        instance.onplay = onplay
        instance.onpause = onpause
        instance.onwaiting = onwaiting
        instance.onprogress = onprogress
        instance.onended = onended
        // this.play()
      },

      switchVol (audio) {
        const {
          instance = {
            play () {}
          }
        } = this

        instance.src = audio.url
        this.play()
      },

      setTime (time) {
        if (!isNaN(time)) {
          this.instance.currentTime = time
          const canPlay = time < this.instance.duration
          if (this.instance.paused && canPlay) {
            this.play()
          }
        }
      },

      onMoveStartIndicator (e) {
        startX = e.touches[0].pageX
        startWidth = this.$refs.playedBar.clientWidth
        this.needTransit = false
      },

      onMoveIndicator (e) {
        const offsetX = e.touches[0].pageX - startX
        const {progressBar} = this.$refs

        let percentage = (startWidth + offsetX) /
          progressBar.clientWidth

        percentage = percentage > 1 ? 1 : percentage
        percentage = percentage < 0 ? 0 : percentage

        this.updateProgress(percentage)
        this.setTime(percentage * this.instance.duration)
      },

      onMoveEndIndicator () {
        this.needTransit = true
      },

      onprogress () {
        const {instance} = this
        if (this.instance.buffered.length) {
          let buffered =
            instance.buffered.end(instance.buffered.length - 1)
          this.buffered =
            buffered / instance.duration * 100
        }
      },

      onended () {
        this.$emit('ended')
      },

      onwaiting () {
        this.loading = true
        // this.showMask = true
      },

      onpause () {
        this.playing = false
      },

      onplay () {
        this.playing = true
        this.showMask = false
      },

      onerror (e) {
        console.warn('Error', this.instance.error)
        this.showMask = true
        // this.errMsg = this.instance.error.message || '加载失败'
        this.errMsg = '加载失败'
      },

      oncanplay () {
        this.loading = false
        this.showMask = false
        this.endTime = UTILS.convertSeconds(this.instance.duration)
        this.$emit('canplay')
      },

      ontimeupdate () {
        const {instance} = this
        this.currentTime = UTILS.convertSeconds(instance.currentTime)
        this.updateProgress(instance.currentTime / instance.duration)
      },

      onClickProgress (e) {
        const {progressBar} = this.$refs
        let percentage = e.offsetX / progressBar.clientWidth
        this.updateProgress(percentage)
        this.setTime(percentage * this.instance.duration)
      },

      updateProgress (percentage) {
        this.progress = percentage * 100
      },

      play () {
        const {instance, loading} = this
        if (!loading && !instance.error) {
          instance.play()
          this.playing = true
        }
      },

      pause () {
        this.instance.pause()
        this.playing = false
      },

      toggle () {
        if (this.playing) {
          this.pause()
        } else {
          this.play()
        }
      },

      destroyInstance () {
        this.pause()
        this.instance.onprogress = ''
        this.instance = ''
        delete this.instance
      }
    }
  }
</script>

<style scoped lang="scss">
  .audio-player {
    position: relative;
    width: 100%;
    background-repeat: no-repeat;
    background-size: cover;
    background-position: center;
    padding-bottom: 56%;
    .audio-controller {
      position: absolute;
      bottom: 0;
      box-sizing: border-box;
      width: 100%;
      padding: 15px 14px;
      color: #fff;
      background: rgba(0, 0, 0, 0.1);
      background: linear-gradient(
        to top,
        rgba(0, 0, 0, 0.3),
        rgba(255, 255, 255, 0.1)
      );
      background: -webkit-linear-gradient(
        bottom,
        rgba(0, 0, 0, 0.3),
        rgba(255, 255, 255, 0.1)
      );

      .audio-time {
        font-size: 12px;
        margin-left: 10px;
        transform: scale(0.833);
        text-align:center;
      }
      .audio-controller-toggle {
        width: 17px;
      }
    }
  }

  .player-mask, .player-mask-loading {
    position: absolute;
    top: 0; left: 0; right: 0; bottom: 0;
    color: #fff;
  }

  .player-mask {
    background-color: rgba(0, 0, 0, 0.3);
  }

  .audio-progress {
    margin-left: 10px;
    height: 2px;
    position: relative;
    background: rgba(0, 0, 0, 0.2);
    overflow: visible;
    .audio-progress-bar {
      position: absolute;
      top: 0;
      left: 0;
      width: 0;
      height: 100%;
      overflow: visible;
      &.ease {
        transition: all 0.5s ease;
      }
      &.audio-progress-bar-played {
        z-index: 2;
        background-color: #fff;
      }
      &.audio-progress-bar-buffered {
        background-color: rgba(0, 0, 0, 0.3);
        transition: all 0.5s ease;
        z-index: 1;
      }
    }
    .audio-indicator {
      display: inline-block;
      position: absolute;
      top: -13px;
      right: -10px;
      cursor: pointer;
      width: 12px;
      height: 12px;
      padding: 8px;
      .audio-indicator-inner {
        width: 12px;
        height: 12px;
        border-radius: 50%;
        background: #fff;
      }
    }
  }

  .audio-player-img {
    width: 100%;
  }

  /* Flex */
  [flex~="cross:center"] {
    -webkit-box-align: center;
    -webkit-align-items: center;
    -ms-flex-align: center;
    -ms-grid-row-align: center;
    align-items: center;
  }

  [flex~="main:center"] {
    -webkit-box-pack: center;
    -ms-flex-pack: center;
    justify-content: center;
  }

  [flex~="dir:top"] {
    -webkit-box-orient: vertical;
    -webkit-box-direction: normal;
    -ms-flex-direction: column;
    flex-direction: column;
  }

  [flex] {
    display: -webkit-box;
    display: -ms-flexbox;
    display: flex;
  }

  [flex], [flex]>*, [flex]>[flex] {
    overflow: hidden;
  }

  [flex-box="1"] {
    -webkit-box-flex: 1;
    -ms-flex-positive: 1;
    flex-grow: 1;
    -ms-flex-negative: 1;
    flex-shrink: 1;
  }

  [flex]>* {
      display: block;
  }
</style>