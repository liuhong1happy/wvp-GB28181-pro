<template>
  <div class="player-tabs-wrapper" ref="playerWrapper">
    <el-tabs v-if="showTab && playerList.length > 1" v-model="activePlayer" type="card" :stretch="true" @tab-click="changePlayer">
      <el-tab-pane v-for="p in playerList" :key="p.key" :label="p.label" :name="p.key"></el-tab-pane>
    </el-tabs>
    <div class="player-video-area" :style="{ height: showTab ? 'calc(100% - 36px)' : '100%' }">
      <jessibucaPlayer
        v-if="activePlayer === 'jessibuca'"
        ref="jessibuca"
        style="width: 100%; height: 100%;"
        :has-audio="hasAudio"
        :show-button="showButton"
        fluent autoplay live
        @playTimeChange="$emit('playTimeChange', $event)"
        @playStatusChange="$emit('playStatusChange', $event)"
      />
      <rtc-player
        v-if="activePlayer === 'webRTC'"
        ref="webRTC"
        style="width: 100%; height: 100%;"
        :has-audio="hasAudio"
        :show-button="showButton"
        fluent autoplay live
        @playTimeChange="$emit('playTimeChange', $event)"
        @playStatusChange="$emit('playStatusChange', $event)"
      />
      <h265web
        v-if="activePlayer === 'h265web'"
        ref="h265web"
        style="width: 100%; height: 100%;"
        :has-audio="hasAudio"
        :show-button="showButton"
        fluent autoplay live
        @playTimeChange="$emit('playTimeChange', $event)"
        @playStatusChange="$emit('playStatusChange', $event)"
      />
    </div>
  </div>
</template>

<script>
import jessibucaPlayer from './jessibuca.vue'
import rtcPlayer from './rtcPlayer.vue'
import h265web from './h265web.vue'

export default {
  name: 'PlayerTabs',
  components: { jessibucaPlayer, rtcPlayer, h265web },
  props: {
    hasAudio: { type: Boolean, default: false },
    showButton: { type: Boolean, default: true },
    showTab: { type: Boolean, default: true },
    // 覆盖默认的「播放器 → 取哪个地址字段」映射（见 data.player）。
    // 只需给出要覆盖的键，其余沿用默认。用于让某个播放器在特定场景取不同的流地址，
    // 例如回放页让 h265web 取 fmp4 而不是 ws_flv（见 cloudRecord/player.vue 的说明）。
    playerUrlFields: { type: Object, default: null }
  },
  data() {
    return {
      streamInfo: null,
      activePlayer: 'jessibuca',
      player: { jessibuca: ['ws_flv', 'wss_flv'], webRTC: ['rtc', 'rtcs'], h265web: ['ws_flv', 'wss_flv'] },
      allPlayerList: [
        { key: 'jessibuca', label: 'Jessibuca' },
        { key: 'webRTC', label: 'WebRTC' },
        { key: 'h265web', label: 'H265web' }
      ]
    }
  },
  computed: {
    playerList() {
      return this.allPlayerList
    },
    playerCount() {
      return this.playerList.length
    }
  },
  created() {
    if (this.playerCount === 1) {
      this.activePlayer = this.playerList[0].key
    }
  },
  methods: {
    getPlayerList() {
      return this.playerList
    },
    getActivePlayer() {
      return this.activePlayer
    },
    switchPlayer(key) {
      if (this.activePlayer === key) return
      this.activePlayer = key
      if (this.streamInfo) {
        this.play()
      }
    },
    getUrlByStreamInfo() {
      if (!this.streamInfo) return ''
      // playerUrlFields 可覆盖默认映射（只覆盖给出的键）
      const fieldsMap = Object.assign({}, this.player, this.playerUrlFields || {})
      const fields = fieldsMap[this.activePlayer]
      if (!fields) return ''
      if (location.protocol === 'https:') {
        // 后端只在 sslPort > 0 时才生成 https_* / wss_* 变体（见 StreamInfo.setWsFlv）。
        // 媒体节点若是通过 auto-config 从 ZLM 同步端口，而 ZLM 的 [http] sslport 为 0
        // （例如它与 Traefik 共用宿主机、443 被占用的情况），那些字段就是 null，
        // 直接取会拿到 undefined，播放器什么也播不出来。
        // 此时退回非加密变体是安全的：经 nginx 的 sub_filter 改写后它与加密变体同为
        // 同源根相对路径（如 /rtp/xxx.live.flv），浏览器会按页面协议自行解析。
        return this.streamInfo[fields[1]] || this.streamInfo[fields[0]] || ''
      }
      return this.streamInfo[fields[0]] || ''
    },
    changePlayer(tab) {
      this.activePlayer = tab.name
      this.play()
      this.$emit('player-changed', this.activePlayer)
    },
    setStreamInfo(streamInfo) {
      this.streamInfo = streamInfo
      this.play()
    },
    play() {
      let playUrl = this.getUrlByStreamInfo()
      this.$nextTick(() => {
        if (this.$refs[this.activePlayer]) {
          this.$refs[this.activePlayer].play(playUrl)
        }
      })
      const typeMap = { jessibuca: 0, webRTC: 1, h265web: 2 }
      const type = typeMap[this.activePlayer] || 0
      const playerUrl = window.location.origin + '/#/play/share?type=' + type + '&url=' + encodeURIComponent(playUrl)
      this.$emit('playerChanged', { playUrl, playerUrl })
    },
    stop() {
      if (this.$refs[this.activePlayer]) {
        this.$refs[this.activePlayer].pause()
      }
    },
    pause() {
      if (this.$refs[this.activePlayer]) {
        this.$refs[this.activePlayer].pause()
      }
    },
    destroy() {
      const player = this.$refs[this.activePlayer]
      if (player && player.destroy) {
        player.destroy()
      }
    },
    setPlaybackRate(rate) {
      const player = this.$refs[this.activePlayer]
      if (player && player.setPlaybackRate) {
        player.setPlaybackRate(rate)
      }
    },
    resize(width, height) {
      const player = this.$refs[this.activePlayer]
      if (player && player.resize) {
        player.resize(width, height)
      }
    },
    screenshot() {
      const player = this.$refs[this.activePlayer]
      if (player && player.screenshot) {
        return player.screenshot()
      }
    },
    getVideoRect() {
      const player = this.$refs[this.activePlayer]
      return player && player.getVideoRect ? player.getVideoRect() : null
    },
    startDragZoom(callback) {
      const player = this.$refs[this.activePlayer]
      if (player && player.startDragZoom) {
        player.startDragZoom(callback)
      }
    }
  }
}
</script>

<style scoped>
.player-tabs-wrapper {
  width: 100%;
  height: 100%;
}
.player-tabs-wrapper .el-tabs {
  margin-bottom: 0;
}
.player-tabs-wrapper .el-tabs >>> .el-tabs__header {
  margin-bottom: 0;
}
.player-video-area {
  width: 100%;
  height: 100%;
  background: #000;
}
</style>
