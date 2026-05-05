+++
date = '2026-05-05T19:28:03+08:00'
draft = false
title = '视频刷新oss key踩坑记录'
+++

## 项目背景：
当前开发的是一个在线课程学习的web应用，引入了阿里云oss对课程视频等文件进行存储，为保证安全性设计后端发放临时url，前端快到期后重新获取并刷新url。

当前项目前端使用的是`vue3`，引入了`videojs`和`@videojs-player/vue`实现播放器。
<!--more-->

## 项目问题
### 临时url刷新失败
编写了相关代码后进行测试，发现临时url不能正常刷新，到达过期时间后拖动到未缓存的地方依然不能正常播放。

### 排查问题
在发现问题时，首先对网络进行抓包，发现有正常发送刷新url请求并获取数据，然后直接访问临时url也可以正常访问，排除了后端的问题；

然后使用DevTools对vue3变量进行访问，发现对应ref内数据**有正常更换**，再对视频加载的网络抓包，发现虽然ref数据正常更换了，但是播放器内部请求使用的url**没有更改**。

### 解决
问题发现后查找资料，了解到播放器内部直接修改引用并**不会触发src响应式更新**，依然是使用初始src进行访问。

针对这个问题，需要获取到`VideoPlayer`实例，调用对应`src`方法。但是获取到的ref实例并不是需要的实例，针对这点，编写了一个测试页面用来观察和寻找`VideoPlayer`实例。
最后发现播放器实例由`@mounted`事件进行返回。

最后修改代码，通过`src`方法实现视频源的正常切换，同时编写了相同视频源和不同视频源的切换逻辑，封装的组件如下：

```vue
<!-- components/VideoPlayer.vue -->
<template>
  <div class="w-full mx-auto bg-black rounded-lg overflow-hidden">
    <!-- 添加 v-if 确保 src 存在再渲染 -->
    <VideoPlayer v-if="props.src" ref="videoPlayerRef" class="video-js vjs-custom-skin" :options="playerOptions"
      :playsinline="playsinline" @mounted="onPlayerMounted" @play="onPlayerPlay" @pause="onPlayerPause"
      @ended="onPlayerEnded" @error="onPlayerError" />

    <div v-else class="p-5 text-red-400 text-center">
      ⚠️ 错误：未传入视频源（src）
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onBeforeUnmount, watch } from 'vue'
import { VideoPlayer } from '@videojs-player/vue'
import type { VideoJsPlayer, VideoJsPlayerOptions } from 'video.js'

interface Props {
  src: string
  poster?: string
  autoplay?: boolean
  muted?: boolean
  controls?: boolean
  loop?: boolean
  playsinline?: boolean
  showCustomControls?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  src: '',
  poster: '',
  autoplay: false,
  muted: false,
  controls: true,
  loop: false,
  playsinline: true,
  showCustomControls: false
})

const videoPlayerRef = ref<InstanceType<typeof VideoPlayer> | null>(null)
const isMuted = ref(props.muted)
// 存储播放器实例
const playerInstance = ref<VideoJsPlayer | null>(null)

// 使用 computed 确保响应式更新
const playerOptions = computed<VideoJsPlayerOptions>(() => ({
  autoplay: props.autoplay,
  controls: props.controls,
  loop: props.loop,
  muted: props.muted,
  language: 'zh-CN',
  preload: 'auto' as const,
  fluid: true,
  playbackRates: [0.5, 1.0, 1.5, 2.0],
  aspectRatio: '16:9',
  notSupportedMessage: '此视频暂无法播放，请稍后再试',
  sources: props.src ? [{
    type: getVideoType(props.src),
    src: props.src
  }] : [],
  poster: props.poster,
  controlBar: {
    timeDivider: true,
    durationDisplay: true,
    remainingTimeDisplay: false,
    fullscreenToggle: true,
    playbackRateMenuButton: true
  }
}))

// 添加空值检查
function getVideoType(src: string): string {
  if (!src) return 'video/mp4'

  const extension = src.split('.').pop()?.toLowerCase()
  const typeMap: Record<string, string> = {
    mp4: 'video/mp4',
    m3u8: 'application/x-mpegURL',
    webm: 'video/webm',
    ogg: 'video/ogg',
    mov: 'video/quicktime',
    flv: 'video/x-flv'
  }
  return typeMap[extension || ''] || 'video/mp4'
}

// 事件处理 - @mounted 事件传递 { video, player, state }
const onPlayerMounted = ({ player }: { player: VideoJsPlayer }) => {
  console.log('✅ 播放器已挂载', player)
  // 保存播放器实例
  playerInstance.value = player
}

const onPlayerPlay = () => {
  console.log('▶️ 开始播放')
}

const onPlayerPause = () => {
  console.log('⏸️ 暂停')
}

const onPlayerEnded = () => {
  console.log('⏹️ 播放结束')
}

const onPlayerError = (error: any) => {
  console.error('❌ 错误:', error)
}

// 控制方法（使用保存的播放器实例）
const play = () => {
  playerInstance.value?.play()
}

const pause = () => {
  playerInstance.value?.pause()
}

const toggleMute = () => {
  if (playerInstance.value) {
    isMuted.value = !isMuted.value
    playerInstance.value.muted(isMuted.value)
  }
}

const toggleFullscreen = () => {
  if (playerInstance.value) {
    playerInstance.value.isFullscreen() ? playerInstance.value.exitFullscreen() : playerInstance.value.requestFullscreen()
  }
}

// 监听 src 变化 - 智能区分"切换视频"和"刷新URL"
watch(() => props.src, (newSrc, oldSrc) => {
  if (!newSrc || newSrc === oldSrc) return

  console.log('[VideoPlayer] 检测到视频URL变化')
  console.log('[VideoPlayer] 旧URL:', oldSrc?.substring(0, 80))
  console.log('[VideoPlayer] 新URL:', newSrc.substring(0, 80))

  if (!playerInstance.value) {
    console.warn('[VideoPlayer] ⚠️ 播放器实例不存在')
    return
  }

  const player = playerInstance.value
  
  // 判断是"切换视频"还是"刷新URL"
  // 如果基础URL相同（忽略查询参数），则是刷新URL；否则是切换视频
  const isSameVideo = oldSrc && getBaseUrl(oldSrc) === getBaseUrl(newSrc)
  
  console.log(`[VideoPlayer] 操作类型: ${isSameVideo ? '🔄 刷新URL（保持进度）' : '🎬 切换视频（重置进度）'}`)

  // 记录当前播放状态
  const wasPlaying = !player.paused()
  const currentTime = player.currentTime() || 0
  const playbackRate = player.playbackRate()

  console.log('[VideoPlayer] 当前状态 - 播放中:', wasPlaying, '进度:', currentTime.toFixed(1), '秒')

  // 暂停播放
  player.pause()

  // 直接设置新源，不先清空（避免触发 MEDIA_ERR_SRC_NOT_SUPPORTED 错误）
  player.src({
    type: getVideoType(newSrc),
    src: newSrc
  })
  player.load()

  // 监听元数据加载完成，恢复播放状态
  player.one('loadedmetadata', () => {
    console.log('[VideoPlayer] 元数据加载完成')

    if (isSameVideo) {
      // 🔄 刷新URL：保持播放进度
      if (currentTime > 0) {
        player.currentTime(currentTime)
        console.log(`[VideoPlayer] ✅ 已恢复播放进度: ${currentTime.toFixed(1)}s`)
      }
    } else {
      // 🎬 切换视频：重置到开头
      player.currentTime(0)
      console.log('[VideoPlayer] ✅ 已重置播放进度到开头')
    }

    // 恢复播放速率
    player.playbackRate(playbackRate)

    // 如果之前在播放，则恢复播放
    if (wasPlaying) {
      const playPromise = player.play()
      if (playPromise) {
        playPromise.catch((err: any) => {
          console.warn('[VideoPlayer] 自动恢复播放失败:', err.message)
        })
      }
      console.log('[VideoPlayer] ✅ 已恢复播放')
    } else {
      console.log('[VideoPlayer] ✅ URL切换完成（保持暂停状态）')
    }
  })

  // 错误处理（忽略清空源时的短暂错误）
  player.one('error', (err: any) => {
    // 如果是清空源导致的错误，忽略它
    if (err.code === 4 && !player.currentSrc()) {
      console.log('[VideoPlayer] 忽略清空源的临时错误')
      return
    }
    console.error('[VideoPlayer] ❌ URL切换失败:', err)
  })
})

// 获取基础URL（去除查询参数）
const getBaseUrl = (url: string): string => {
  try {
    const urlObj = new URL(url)
    return `${urlObj.protocol}//${urlObj.host}${urlObj.pathname}`
  } catch {
    return url.split('?')[0] || ''
  }
}

onBeforeUnmount(() => {
  playerInstance.value?.dispose()
  playerInstance.value = null
})

defineExpose({
  play,
  pause,
  toggleMute,
  toggleFullscreen,
  getPlayer: () => playerInstance.value
})
</script>

<style scoped>
/* 这些样式是必需的，因为它们是作用于第三方 Video.js 组件的内部元素，
   Tailwind 无法直接穿透组件边界影响子组件内部元素 */
:deep(.video-js .vjs-big-play-button) {
  top: 50% !important;
  left: 50% !important;
  transform: translate(-50%, -50%) !important;
  margin: 0 !important;
  position: absolute !important;
}

:deep(.video-js) {
  position: relative !important;
}
</style>
```

### 小插曲
在后端编写了相关api后我先对临时url的使用进行了测试。测试结果发现临时url可以正常使用，但是过期后依然可以观看。
在查阅资料后了解播放器会对视频进行分段缓存，而可以观看的部分是已经本地缓存的部分，不影响设计。

---
**总结**

针对这次视频组件踩坑记录，了解相关组件切换源的问题和正确的使用方法，也对视频播放相关逻辑有了更深的理解。