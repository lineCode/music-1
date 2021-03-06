<template>
    <div :class="{[s.app]:true}">
        <div :class="s.wrap"></div>
        <div :class="s.cover" :style="style"></div>
        <div :class="s.control">
            <div :class="s.inner" v-if="status!=='fullscreen'">
                <Icon type="bottom" :class="s.close" @click.native="close"></Icon>
                <Icon type="fullscreen" :class="s.fullscreen" @click.native="op('fullscreen')"></Icon>
                <Icon type="narrow" :class="s.narrow" @click.native="op('minimize')"></Icon>
            </div>
            <!-- 全屏状态只有离开全屏 !-->
            <Icon type="fullscreenexit" :class="s.fullscreen" @click.native="op('leaveFullscreen')" v-else></Icon>
        </div>
        <ul :class="s.main" ref="main" @wheel="scrollBarWheel" v-if="lyrics.length">
            <li v-for="(item,index) in lyrics" :class="{[s.item]:true,[s.active]:activeIndex === index}">
                {{item[1]}}
            </li>
        </ul>
        <div :class="s.main" v-else>
            <span :class="[s.item,s.nolyric,s.active]">暂无歌词信息...</span>
        </div>
    </div>
</template>
<script>
  import { mapState } from 'vuex'
  import Velocity from 'velocity-animate'

  export default {
    data() {
      return {
        userScrolling: false,
        timer: null,
      }
    },
    computed: {
      ...mapState('lyrics', ['show', 'loading', 'lyrics', 'activeIndex']),
      ...mapState('api', ['play']),
      ...mapState('windowStatus', ['status']),
      style() {
        if (this.play.info) {
          return {
            'background-image': `url(${this.play.info.album.cover})`
          }
        } else {
          return {}
        }
      },
    },
    watch: {
      'play.info'() {
        console.log('change')
        if (this.$refs.main) {
          this.$refs.main.scrollTop = 0
        }
      },
      'activeIndex'(val) {
        const main = this.$refs.main
        if (main && main.children[val]) {
          // 传递到状态栏
          this.transfer(val)
          const need = main.children[val].offsetTop - main.offsetHeight / 2
          // 是否打开了歌词面板
          if (this.show) {
            // 判断是否用户处于滚动浏览中
            if (!this.userScrolling) {
              if (need !== main.scrollTop) {
                Velocity(main, 'stop')
                Velocity(main, 'scroll', {
                  container: main,
                  duration: 300,
                  offset: need - main.scrollTop
                })
              }
            }
          } else { // 未打开的话则不用缓动歌词
            Velocity(main, 'stop')
            main.scrollTop = need
          }
        }
      }
    },
    methods: {
      close() {
        this.$store.commit('lyrics/update', {
          show: false
        })
      },
      scrollBarWheel() {
        this.userScrolling = true
        if (this.timer) {
          window.clearTimeout(this.timer)
        }
        this.timer = window.setTimeout(() => {
          const main = this.$refs.main
          main.scrollTop = main.children[this.activeIndex].offsetTop - main.offsetHeight / 2
          this.userScrolling = false
        }, 2000)
      },
      op(val) {
        this.$store.commit('windowStatus/update', val)
      },
      async transfer(index) {
        const lyric = this.lyrics
        const singleLyric = lyric[index][1] // 单句歌词
        // 先计算当前句 和 下一句 的时间差
        let time = 0
        if (index === lyric.length - 1) { // 如果是最后一行 就默认10s吧，没有可靠的逻辑
          time = 10 * 1000
        } else {
          time = lyric[index + 1][0] - lyric[index][0]
        }
        // 再计算一下总字符长度 决定了要截断几次
        let totalLength = 0
        for (let i = 0; i < singleLyric.length; i++) {
          if (singleLyric.charCodeAt(i) > 127 || singleLyric.charCodeAt(i) === 94) {
            totalLength += 2
          } else {
            totalLength += 1
          }
        }
        // 开始截断
        let str = ''
        let len = 0
        let singleLen = 0
        for (let i = 0; i < singleLyric.length; i++) {
          if (singleLyric.charCodeAt(i) > 127 || singleLyric.charCodeAt(i) === 94) {
            singleLen = 2
          } else {
            singleLen = 1
          }
          if (len + singleLen > 26) { // 超出 直接传递到控制台
            this.$ipc.send('tray-control-lyrics', str)
            str = singleLyric[i]
            len = singleLen
            await new Promise((resolve) => {
              window.setTimeout(() => resolve(), parseInt(time * 26 / totalLength)) // 延时
            })
          } else {
            str += singleLyric[i]
            len += singleLen
            if (i === singleLyric.length - 1) {
              this.$ipc.send('tray-control-lyrics', str)
            }
          }
        }
      }
    }
  }
</script>
<style lang="scss" module="s">
    .app {
        position: fixed;
        z-index: 5;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        padding-bottom: 60px;
        will-change: transform;
        transition: all .4s;
        .wrap,
        .cover,
        .main {
            position: absolute;
            left: 0;
            top: 0;
            width: 100%;
            height: 100%;
        }
        .wrap {
            z-index: 4;
            background-color: black;
        }
        .cover {
            z-index: 5;
            background-repeat: no-repeat;
            background-position: center;
            background-size: cover;
            filter: blur(35px);
            opacity: .3;
        }
        .main {
            z-index: 6;
            text-align: center;
            color: rgba(255, 255, 255, 0.9);
            overflow: auto;
            height: calc(100% - 60px);
            .item {
                padding: 8px 0;
                &.active {
                    color: #26B36C;
                }
            }
            .nolyric {
                display: inline-flex;
                margin-top: 20px;
            }
            &::-webkit-scrollbar {
                display: none;
            }
        }
        .control {
            position: absolute;
            left: 16px;
            top: 12px;
            background-color: transparent;
            z-index: 7;
            width: 200px;
            height: 50px;
            -webkit-app-region: no-drag;
            .inner {
                display: flex;
            }
            svg {
                font-size: 24px;
                margin-right: 8px;
                color: #B7B7B7;
                &:hover {
                    color: #f2f2f2;
                }
            }
        }
    }
</style>