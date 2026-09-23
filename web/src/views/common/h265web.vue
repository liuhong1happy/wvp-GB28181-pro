<template>
  <div :id="'h265Player-' + _uid" ref="container" style="background-color: #000000; position: relative; display: flex; align-items: center; justify-content: center;" @dblclick="fullscreenSwich" @mouseenter="showBar = true" @mouseleave="showBar = false">
    <div :id="'glplayer-' + _uid" ref="playerBox" style="width: 100%; height: 100%; margin: 0 auto;">
      <div v-if="playerLoading" class="play-loading">
        <i class="el-icon-loading" />
        <span style="margin-left: 5px">视频加载中</span>
      </div>
    </div>

    <div v-if="showButton" id="buttonsBox" class="buttons-box" :style="{ opacity: showBar ? 1 : 0, pointerEvents: showBar ? 'auto' : 'none' }">
      <div class="buttons-box-left">
        <i v-if="!playing" class="iconfont icon-play h265web-btn" @click="unPause" />
        <i v-if="playing" class="iconfont icon-pause h265web-btn" @click="pause" />
        <i class="iconfont icon-stop h265web-btn" @click="destroy" />
        <i v-if="isNotMute" class="iconfont icon-audio-high h265web-btn" @click="mute()" />
        <i v-if="!isNotMute" class="iconfont icon-audio-mute h265web-btn" @click="cancelMute()" />
      </div>
      <div class="buttons-box-right">
        <!--          <i class="iconfont icon-file-record1 h265web-btn"></i>-->
        <!--          <i class="iconfont icon-xiangqing2 h265web-btn" ></i>-->
        <i
          class="iconfont icon-camera1196054easyiconnet h265web-btn"
          style="font-size: 1rem !important"
          @click="screenshot"
        />
        <i class="iconfont icon-shuaxin11 h265web-btn" @click="playBtnClick" />
        <i v-if="!fullscreen" class="iconfont icon-weibiaoti10 h265web-btn" @click="fullscreenSwich" />
        <i v-if="fullscreen" class="iconfont icon-weibiaoti11 h265web-btn" @click="fullscreenSwich" />
      </div>
    </div>
  </div>
</template>

<script>
const h265webPlayer = {}
/**
 * 新版 SDK（v20260824）的用法，详见上游 DOCS/README-HowToUse-CN.MD：
 *   H265webjsPlayer() -> build(config) -> load_media(url)
 * 旧版是 `new window.new265webjs(url, opts)` 且需要 token，新版不再需要。
 * wasm / ext 四个运行时文件由 config 里的 *_uri 按需加载，index.html 只需引入 h265web.js。
 */
const SDK_BASE_URL = './static/js/h265web/' // 相对站点根，dist 里就是 /static/js/h265web/
import dragZoom from '../../mixins/dragZoom'
export default {
  name: 'H265web',
  mixins: [dragZoom],
  props: ['videoUrl', 'error', 'hasAudio', 'height', 'showButton'],
  data() {
    return {
      playing: false,
      isNotMute: false,
      quieting: false,
      fullscreen: false,
      loaded: false, // mute
      speed: 0,
      kBps: 0,
      btnDom: null,
      videoInfo: null,
      volume: 1,
      rotate: 0,
      vod: true, // 点播
      forceNoOffscreen: false,
      playerWidth: 0,
      playerHeight: 0,
      inited: false,
      playerLoading: false,
      mediaInfo: null,
      showBar: true,
      err: ''
    }
  },
  watch: {
    playing(newData, oldData) {
      this.$emit('playStatusChange', newData)
    },
    immediate: true
  },
  mounted() {},
  destroyed() {
    this.destroy()
  },
  methods: {
    updatePlayerDomSize() {
      const dom = this.$refs.container
      if (!this.parentNodeResizeObserver) {
        this.parentNodeResizeObserver = new ResizeObserver(entries => {
          this.updatePlayerDomSize()
        })
        this.parentNodeResizeObserver.observe(dom.parentNode)
      }
      const boxWidth = dom.parentNode.clientWidth
      const boxHeight = dom.parentNode.clientHeight
      let width = boxWidth
      let height = (9 / 16) * width
      if (boxHeight > 0 && boxWidth > boxHeight / 9 * 16) {
        height = boxHeight
        width = boxHeight / 9 * 16
      }

      const clientHeight = Math.min(document.body.clientHeight, document.documentElement.clientHeight)
      if (height > clientHeight) {
        height = clientHeight
        width = (16 / 9) * height
      }

      this.$refs.playerBox.style.width = width + 'px'
      this.$refs.playerBox.style.height = height + 'px'
      this.playerWidth = width
      this.playerHeight = height
      if (this.playing) {
        h265webPlayer[this._uid].resize(this.playerWidth, this.playerHeight)
      }
    },
    resize(width, height) {
      this.playerWidth = width
      this.playerHeight = height
      this.$refs.playerBox.style.width = width + 'px'
      this.$refs.playerBox.style.height = height + 'px'
      if (this.playing) {
        h265webPlayer[this._uid].resize(this.playerWidth, this.playerHeight)
      }
    },
    create(url) {
      if (typeof window.H265webjsPlayer !== 'function') {
        this.playerLoading = false
        this.err = 'h265web.js 未加载，检查 index.html 的 script 引入'
        console.error(this.err)
        return
      }
      this.playerLoading = true
      const player = window.H265webjsPlayer()

      // 回调需在 build() 之前挂上
      player.on_ready_show_done_callback = () => {
        // 画面就绪，尝试自动播放
        this.playing = !!player.play()
        this.playerLoading = false
      }
      player.video_probe_callback = (mediaInfo) => {
        // 探测完成，拿到媒体信息（替代旧版的 onLoadFinish + mediaInfo()）
        this.loaded = true
        this.mediaInfo = mediaInfo
      }
      player.on_play_time = (videoPTS) => {
        this.$emit('playTimeChange', videoPTS * 1000)
      }
      player.on_play_finished = () => {
        this.playing = false
      }
      player.on_error_callback = (err) => {
        this.playerLoading = false
        this.playing = false
        this.err = err ? (err.message || err.msg || String(err)) : '播放出错'
        console.error('h265web 播放错误', err)
      }

      const buildResult = player.build({
        player_id: 'glplayer-' + this._uid,
        base_url: SDK_BASE_URL,
        wasm_js_uri: 'h265web_wasm.js',
        wasm_wasm_uri: 'h265web_wasm.wasm',
        ext_src_js_uri: 'extjs.js',
        ext_wasm_js_uri: 'extwasm.js',
        width: this.playerWidth,
        height: this.playerHeight,
        color: '#000000',
        auto_play: false, // 交给 on_ready_show_done_callback 显式播放，与旧版行为一致
        ignore_audio: this.hasAudio === null ? false : !this.hasAudio
      })
      if (!buildResult) {
        this.playerLoading = false
        this.err = 'h265web build 失败'
        console.error(this.err)
        return
      }

      h265webPlayer[this._uid] = player
      player.load_media(url)
    },
    screenshot: function() {
      const player = h265webPlayer[this._uid]
      if (!player) return
      // 新版 SDK 是 screenshot(元素id)，把画面画进一个已存在的 <img>，
      // 不再像旧版那样 snapshot(canvas)，所以这里自建一个隐藏 img 来接。
      const imgId = 'h265web-shot-' + this._uid
      let img = document.getElementById(imgId)
      if (!img) {
        img = document.createElement('img')
        img.id = imgId
        img.style.display = 'none'
        document.body.appendChild(img)
      }
      player.screenshot(imgId)
      // 绘制是否同步上游未说明，留一拍再取，拿不到就只告警不报错
      setTimeout(() => {
        if (!img.src) {
          console.warn('h265web 截图未返回图像')
          return
        }
        const link = document.createElement('a')
        link.download = 'screenshot.png'
        link.href = img.src
        link.click()
      }, 300)
    },
    playBtnClick: function(event) {
      this.play(this.videoUrl)
    },
    refresh: function() {
      this.play(this.videoUrl)
    },
    play: function(url) {
      if (h265webPlayer[this._uid]) {
        this.destroy()
      }
      if (!url) {
        return
      }
      if (this.playerWidth === 0 || this.playerHeight === 0) {
        this.updatePlayerDomSize()
        setTimeout(() => {
          this.play(url)
        }, 300)
        return
      }
      this.create(url)
    },
    unPause: function() {
      if (h265webPlayer[this._uid]) {
        h265webPlayer[this._uid].play()
        this.playing = h265webPlayer[this._uid].isPlaying()
      }
      this.err = ''
    },
    pause: function() {
      if (h265webPlayer[this._uid]) {
        h265webPlayer[this._uid].pause()
        this.playing = h265webPlayer[this._uid].isPlaying()
      }
      this.err = ''
    },
    mute: function() {
      if (h265webPlayer[this._uid]) {
        // 新版：set_voice(0) 静音，set_voice(正值) 恢复
        h265webPlayer[this._uid].set_voice(0.0)
        this.isNotMute = false
      }
    },
    cancelMute: function() {
      if (h265webPlayer[this._uid]) {
        h265webPlayer[this._uid].set_voice(this.volume || 1.0)
        this.isNotMute = true
      }
    },
    destroy: function() {
      if (h265webPlayer[this._uid]) {
        h265webPlayer[this._uid].release()
      }
      h265webPlayer[this._uid] = null
      this.playing = false
      this.err = ''
    },
    stop: function() {
      this.destroy()
    },
    fullscreenSwich: function() {
      const isFull = this.isFullscreen()
      if (isFull) {
        h265webPlayer[this._uid].closeFullScreen()
      } else {
        h265webPlayer[this._uid].fullScreen()
      }
      this.fullscreen = !isFull
    },
    isFullscreen: function() {
      return document.fullscreenElement ||
        document.msFullscreenElement ||
        document.mozFullScreenElement ||
        document.webkitFullscreenElement || false
    },
    setPlaybackRate: function(speed) {
      if (h265webPlayer[this._uid]) {
        h265webPlayer[this._uid].set_playback_rate(speed)
      }
    },
    getVideoElement() {
      return this.$refs.playerBox
    },
    getVideoRect() {
      return this.getVideoElement().getBoundingClientRect()
    }
  }
}
</script>

<style>
.play-loading {
  width: 100%;
  height: 100%;
  color: rgb(255, 255, 255);
  display: flex;
  align-items: center;
  margin: 0 auto;
  justify-content: center;
  font-size: 18px;
}
.buttons-box {
  width: 100%;
  height: 56px;
  background: linear-gradient(to top, rgba(0, 0, 0, 1), rgba(0, 0, 0, 0));
  position: absolute;
  transition: opacity 0.3s ease;
  display: -webkit-box;
  display: -ms-flexbox;
  display: flex;
  align-items: flex-end;
  padding-bottom: 10px;
  left: 0;
  bottom: 0;
  user-select: none;
  z-index: 10;
}

.h265web-btn {
  width: 20px;
  color: rgb(255, 255, 255);
  margin: 0px 10px;
  padding: 0px 2px;
  cursor: pointer;
  text-align: center;
  font-size: 0.8rem !important;
}

.buttons-box-right {
  position: absolute;
  right: 0;
  bottom: 10px;
}
.player-loading {
  width: fit-content;
  height: 30px;
  position: absolute;
  left: calc(50% - 52px);
  top: calc(50% - 52px);
  color: #fff;
  font-size: 16px;
}
.player-loading i{
  font-size: 24px;
  line-height: 24px;
  text-align: center;
  display: block;
}
.player-loading span{
  display: inline-block;
  font-size: 16px;
  height: 24px;
  line-height: 24px;
}
</style>
