<template> 
  <vue-plyr ref="plyr"
    :emit="['canplay', 'timeupdate', 'ended', 'seeked']"
    @canplay="onCanplay()"
    @timeupdate="onTimeupdate()"
    @ended="onEnded()"
    @seeked="onSeeked()"
  >
    <audio crossorigin="anonymous" >
      <source v-if="source" :src="source" />
    </audio>
  </vue-plyr>
</template>

<script>
import Lyric from 'lrc-file-parser'
import { mapState, mapGetters } from 'vuex'

export default {
  name: 'AudioElement',

  data() {
    return {
      lrcObj: null,
      lrcAvailable: false,
    }
  },

  computed: {
    player () {
      return this.$refs.plyr.player
    },

    source () {
      // 从 LocalStorage 中读取 token
      const token = this.$q.localStorage.getItem('jwt-token') || ''
      return this.currentPlayingFile.hash ? `/api/stream/${this.currentPlayingFile.hash}?token=${token}` : ""
    },

    ...mapState('AudioPlayer', [
      'playing',
      'queue',
      'queueIndex',
      'playMode',
      'muted',
      'volume'
    ]),

    ...mapGetters('AudioPlayer', [
      'currentPlayingFile'
    ])
  },

  watch: {
    playing (flag) {  
      if (this.player.duration) {
        // 缓冲至可播放状态
        flag ? this.player.play() : this.player.pause()
      }
      this.playLrc(flag);
    },

    source (url) {
      if (url) {   
        // 加载新音频/视频文件
        this.player.media.load();
        this.loadLrcFile();
      }
    },

    muted (flag) {
      // 切换静音状态
      this.player.muted = flag
    },

    volume (val) {
      // 屏蔽非法数值
      if (val < 0 || val > 1) {
        return
      }
      
      // 调节音量
      this.player.volume = val
    }
  },

  methods: {
    onCanplay () {
      // 缓冲至可播放状态时触发 (只有缓冲至可播放状态, 才能获取媒体文件的播放时长)
      this.$store.commit('AudioPlayer/SET_DURATION', this.player.duration)

      // 播放
      if (this.playing && this.player.currentTime !== this.player.duration) {
        this.player.play()
      } 
    },

    onTimeupdate () {
      // 当目前的播放位置已更改时触发
      this.$store.commit('AudioPlayer/SET_CURRENT_TIME', this.player.currentTime)
    },

    onEnded () {
      // 当前文件播放结束时触发
      switch (this.playMode.name) {
        case "all repeat":
          // 循环播放
          if (this.queueIndex === this.queue.length - 1) {
            this.$store.commit('AudioPlayer/SET_TRACK', 0)
          } else {
            this.$store.commit('AudioPlayer/NEXT_TRACK')
          }
          break
        case "repeat once":
          // 单曲循环
          this.player.currentTime = 0
          this.$store.commit('AudioPlayer/PLAY')
          break
        case "shuffle": {
          // 随机播放
          const index = Math.floor(Math.random()*this.queue.length)
          this.$store.commit('AudioPlayer/SET_TRACK', index)
          if (index === this.queueIndex) {
            this.player.currentTime = 0
          }
          break
        }
        default:
          // 顺序播放
          if (this.queueIndex === this.queue.length - 1) {
            this.$store.commit('AudioPlayer/PAUSE')
          } else {
            this.$store.commit('AudioPlayer/NEXT_TRACK')
          }
      }
    },

    onSeeked() {
      if (this.lrcAvailable) {
        this.lrcObj.play(this.player.currentTime * 1000);
      }
    },


    playLrc (playStatus) {
      if (this.lrcAvailable) {
        if (playStatus) {
          this.lrcObj.play(this.player.currentTime * 1000);
        } else {
          this.lrcObj.pause();
        }
      }
    },

    initLrcObj () {
        this.lrcObj = new Lyric({
          onPlay: (line, text) => {
            this.$store.commit('AudioPlayer/SET_CURRENT_LYRIC', text);
          },
        })
    },

    loadLrcFile () {
      const token = this.$q.localStorage.getItem('jwt-token') || '';
      const fileHash = this.queue[this.queueIndex].hash;
      const url = `/api/check-lrc/${fileHash}?token=${token}`;

      this.$axios.get(url)
        .then((response) => {
          if (response.data.result) {
            // 有lrc歌词文件
            this.lrcAvailable = true;
            console.log('读入歌词');
            const lrcUrl = `/api/stream/${response.data.hash}?token=${token}`;
            this.$axios.get(lrcUrl)
              .then(response => {
                console.log('歌词读入成功');
                this.lrcObj.setLyric(response.data);
                this.lrcObj.play(this.player.currentTime * 1000);
              });
          } else {
            // 无歌词文件
            this.lrcAvailable = false;
            this.lrcObj.setLyric('');
            let dom_lyric = document.getElementById('lyric');
            if (dom_lyric) {
              dom_lyric.innerHTML = '';
            }
          }
        })
        .catch((error) => {
          if (error.response) {
            // 请求已发出，但服务器响应的状态码不在 2xx 范围内
            if (error.response.status !== 401) {
              this.showErrNotif(error.response.data.error || `${error.response.status} ${error.response.statusText}`);
            }
          } else {
            this.showErrNotif(error.message || error);
          }
        })
    },

    showErrNotif (message) {
      this.$q.notify({
        message,
        color: 'negative',
        icon: 'bug_report'
      })
    }
  },

  mounted () {
    // 初始化音量
    this.$store.commit('AudioPlayer/SET_VOLUME', this.player.volume);
    this.initLrcObj();
    if (this.source) {
      this.loadLrcFile();
    }
  }
}
</script>
