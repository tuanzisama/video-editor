<template>
  <div class="app">
    <header class="app-header">
      <div class="header-brand">
        <svg class="brand-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <rect x="2" y="3" width="20" height="14" rx="2" />
          <path d="M8 21h8M12 17v4" />
          <polyline points="9,9 15,12 9,15" />
        </svg>
        <h1 class="brand-title">Video Editor</h1>
      </div>
      <span class="header-tag">Browser-based</span>
    </header>

    <div class="video-grid">
      <div class="video-card preview">
        <div class="card-top">
          <span class="card-badge preview-badge">预览区</span>
        </div>
        <t-loading class="card-player" :loading="isVideoLoading">
          <video ref="videoRef" :src="rawFileBlobUrl" controls
            @loadeddata="onVideoLoadedDataHandler"></video>
        </t-loading>
      </div>

      <div class="video-card output" :class="{ active: isOutputed }"
        :style="{ '--output-percent': `${generatePercent}%` }">
        <div class="card-top">
          <span class="card-badge output-badge">生成区</span>
          <span v-if="isFFmpegProcessing" class="processing-hint">处理中…</span>
          <span v-else-if="isOutputed" class="done-hint">完成</span>
        </div>
        <t-loading class="card-player" :loading="isFFmpegProcessing">
          <video ref="videoOutoutRef" :src="outputVideoBlobUrl" controls></video>
        </t-loading>
        <div class="progress-track" v-if="isFFmpegProcessing || isOutputed">
          <div class="progress-fill"></div>
        </div>
      </div>
    </div>

    <!-- 音视频编码配置面板（暂未开放） -->
    <div v-if="false" class="settings-section" :class="{ expanded: showSettings }">
      <div class="settings-toggle" @click="showSettings = !showSettings">
        <svg :class="{ rotated: showSettings }" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round">
          <polyline points="6 9 12 15 18 9" />
        </svg>
        <span>编码设置</span>
        <span class="settings-hint" v-if="!needsReEncode">快速剪切模式（无损）</span>
        <span class="settings-hint re-encode" v-else>重编码模式</span>
        <t-button size="small" variant="text" theme="primary" style="margin-left:auto" @click.stop="resetEncodingConfig">重置</t-button>
      </div>
      <div class="settings-body" v-show="showSettings">
        <div class="settings-row">
          <div class="settings-group">
            <div class="settings-group-title">视频</div>
            <div class="settings-field">
              <label class="field-label">编码器</label>
              <t-select v-model="videoCodec" :options="[
                { label: '不重编码（最快）', value: 'copy' },
                { label: 'H.264 (libx264)', value: 'h264' }
              ]" size="small" style="width:100%" />
            </div>
            <div class="settings-field">
              <label class="field-label">分辨率</label>
              <t-select v-model="videoResolution" :options="[
                { label: '原始', value: 'original' },
                { label: '1080p', value: '1080' },
                { label: '720p', value: '720' },
                { label: '480p', value: '480' },
                { label: '360p', value: '360' }
              ]" size="small" style="width:100%" />
            </div>
            <div class="settings-field">
              <label class="field-label">帧率</label>
              <t-select v-model="videoFrameRate" :options="[
                { label: '原始', value: 'original' },
                { label: '60', value: '60' },
                { label: '30', value: '30' },
                { label: '24', value: '24' }
              ]" size="small" style="width:100%" />
            </div>
            <div class="settings-field">
              <label class="field-label">码率 (kbps)</label>
              <t-input-number v-model="videoBitrate" :min="100" :max="100000" :step="500"
                placeholder="自动" size="small" theme="normal" style="width:100%" />
            </div>
            <div class="settings-field">
              <label class="field-label">CRF 质量</label>
              <t-slider v-model="videoCRF" :min="0" :max="51" :step="1" :disabled="videoCodec === 'copy'" />
              <span class="field-hint">{{ videoCRF }} (越低质量越高)</span>
            </div>
          </div>

          <div class="settings-group">
            <div class="settings-group-title">音频</div>
            <div class="settings-field">
              <label class="field-label">编码器</label>
              <t-select v-model="audioCodec" :options="[
                { label: '不重编码', value: 'copy' },
                { label: 'AAC (内置)', value: 'aac' }
              ]" size="small" style="width:100%" />
            </div>
            <div class="settings-field">
              <label class="field-label">码率 (kbps)</label>
              <t-select v-model="audioBitrate" :options="[
                { label: '64', value: 64 },
                { label: '96', value: 96 },
                { label: '128', value: 128 },
                { label: '192', value: 192 },
                { label: '256', value: 256 },
                { label: '320', value: 320 }
              ]" size="small" style="width:100%" />
            </div>
            <div class="settings-field">
              <label class="field-label">采样率</label>
              <t-select v-model="audioSampleRate" :options="[
                { label: '原始', value: 'original' },
                { label: '44100 Hz', value: '44100' },
                { label: '48000 Hz', value: '48000' },
                { label: '22050 Hz', value: '22050' }
              ]" size="small" style="width:100%" />
            </div>
          </div>

          <div class="settings-group">
            <div class="settings-group-title">压缩预设</div>
            <div class="settings-field">
              <label class="field-label">质量阈值</label>
              <t-select v-model="compressionPreset" :options="[
                { label: '视觉无损 (CRF 15)', value: 'visually_lossless' },
                { label: '高质量 (CRF 18)', value: 'high' },
                { label: '中等 (CRF 23)', value: 'medium' },
                { label: '低质量 (CRF 28)', value: 'low' }
              ]" size="small" style="width:100%" @change="onCompressionPresetChange" />
            </div>
            <div class="settings-field">
              <label class="field-label">输出格式</label>
              <t-select v-model="outputFormat" :options="[
                { label: 'MP4', value: 'mp4' },
                { label: 'MKV', value: 'mkv' }
              ]" size="small" style="width:100%" />
            </div>
            <div class="settings-field">
              <label class="field-label">自动格式</label>
              <span class="field-hint">→ {{ outputFormatComputed }}</span>
            </div>
          </div>
        </div>
      </div>
    </div>

    <footer class="control-bar">
      <div class="bar-section bar-file">
        <t-button theme="primary" @click="onSelectMediaFileClickHandler">
          <template #icon><span class="btn-icon">+</span></template>
          选择文件
        </t-button>
        <template v-if="rawFile">
          <span class="bar-meta" :title="rawFile.name">{{ rawFile.name }}</span>
          <span class="bar-sep">·</span>
          <span class="bar-meta">{{ prettyBytes(rawFile.size) }}</span>
          <span class="bar-sep">·</span>
          <span class="bar-meta mono">{{ rawFile.type || 'video/*' }}</span>
        </template>
      </div>

      <div class="bar-section bar-timeline" v-if="isVideoLoaded">
        <div class="tl-node tl-start">
          <span class="tl-time">{{ formatTime(slideRange[0]) }}</span>
          <span class="tl-tag">开始</span>
        </div>
        <div class="tl-track">
          <div class="tl-range-visual" :style="{ left: leftPercent, right: rightPercent }"></div>
          <t-slider v-model="slideRange" :max="videoDuration" range :disabled="isFFmpegProcessing" @change="onSliderChangeHandler" />
        </div>
        <div class="tl-node tl-end">
          <span class="tl-time">{{ formatTime(slideRange[1]) }}</span>
          <span class="tl-tag">结束</span>
        </div>
        <div class="tl-duration">
          <span class="tl-duration-value">{{ clipDurationSeconds }}s</span>
          <span class="tl-duration-label">时长</span>
        </div>
      </div>

      <div class="bar-section bar-actions">
        <t-button theme="success" :disabled="!isVideoLoaded"
          :loading="isFFmpegProcessing" @click="onStartTransCodeClickHandler">
          {{ isFFmpegProcessing ? '正在生成' : '开始生成' }}
        </t-button>
        <t-button variant="outline" :disabled="!isOutputed"
          @click="onDownloadOutputClickHandler">
          下载结果
        </t-button>
        <t-button variant="outline"
          @click="showLogDrawer = true">
          查看日志
        </t-button>
      </div>

      <template v-if="isOutputed">
        <span class="bar-sep pipe">|</span>
        <div class="bar-meta-group">
          <span class="bar-meta-label">用时</span>
          <t-tooltip :content="`精确: ${outputUsageDuration}ms`" placement="top" show-arrow>
            <span class="bar-meta accent">{{ (outputUsageDuration / 1000).toFixed(2) }}s</span>
          </t-tooltip>
        </div>
        <div class="bar-meta-group">
          <span class="bar-meta-label">体积</span>
          <span class="bar-meta">{{ prettyBytes(rawFile!.size) }} → {{ prettyBytes(outputVideoBlob!.size) }}</span>
        </div>
        <div class="bar-meta-group">
          <span class="bar-meta-label">格式</span>
          <span class="bar-meta mono">{{ outputVideoBlob!.type }}</span>
        </div>
      </template>
    </footer>

    <t-drawer v-model:visible="showLogDrawer" :size="600" z-index="4000" :footer="false">
      <template #header>
        <div class="log-drawer-header">
          <span>日志</span>
          <span class="log-count">{{ ffmpegLogs.length }} 条</span>
          <t-button size="small" theme="danger" :disabled="ffmpegLogs.length === 0"
            @click="ffmpegLogs = []">清空</t-button>
        </div>
      </template>
      <div class="log-container">
        <div v-if="ffmpegLogs.length === 0" class="log-empty">暂无日志</div>
        <div v-for="(line, i) in ffmpegLogs" :key="i" class="log-line">{{ line }}</div>
      </div>
    </t-drawer>
  </div>
</template>

<script setup lang="ts">
import { MessagePlugin } from 'tdesign-vue-next';
import { computed, onBeforeMount, onMounted, ref, shallowRef, watch } from 'vue';
import { FFmpeg } from '@ffmpeg/ffmpeg'
import { fetchFile, toBlobURL } from '@ffmpeg/util';
import NotificationPlugin from 'tdesign-vue-next/es/notification/plugin';
import { saveAs } from 'file-saver';

import dayjs from 'dayjs';
import objectSupport from 'dayjs/plugin/objectSupport'

import prettyBytes from 'pretty-bytes';

const videoRef = ref<HTMLVideoElement>()
const videoOutoutRef = ref<HTMLVideoElement>()
const rawFileBlobUrl = ref<string>();
const rawFile = ref<File>()
const slideRange = ref<number[]>([0, 100])
const videoDuration = ref<number>(0)
const isVideoLoading = ref<boolean>(false)
const isVideoLoaded = ref<boolean>(false)
const ffmpegRef = shallowRef<FFmpeg>();

const isFFmpegProcessing = ref<boolean>(false)
const outputVideoBlob = ref<Blob | null>(null)
const outputVideoBlobUrl = ref<string>()
const outputUsageDuration = ref<number>(0)
const generatePercent = ref<string>('0%')
const isOutputed = ref<boolean>(false)

// ===== 日志 =====
const showLogDrawer = ref<boolean>(false)
const ffmpegLogs = ref<string[]>([])

const logUserAction = (action: string) => {
  const ts = new Date().toISOString().slice(11, 23)
  ffmpegLogs.value.push(`[${ts}] [用户] ${action}`)
}

// ===== 音视频编码配置 =====
const showSettings = ref<boolean>(false)
const videoCodec = ref<string>('copy')
const videoResolution = ref<string>('original')
const videoFrameRate = ref<string>('original')
const videoBitrate = ref<number | undefined>(undefined)
const videoCRF = ref<number>(18)
const audioCodec = ref<string>('copy')
const audioBitrate = ref<number>(128)
const audioSampleRate = ref<string>('original')
const outputFormat = ref<string>('mp4')
const compressionPreset = ref<string>('visually_lossless')

const encodingConfig = ref({
  videoCodec, videoResolution, videoFrameRate, videoBitrate, videoCRF,
  audioCodec, audioBitrate, audioSampleRate, outputFormat
})

const outputFormatComputed = computed(() => {
  if (videoCodec.value === 'copy') {
    const ext = rawFile.value?.name.split('.').pop()?.toLowerCase()
    return ext === 'mkv' ? 'mkv' : 'mp4'
  }
  return 'mp4'
})

const onCompressionPresetChange = (val: string) => {
  const presets: Record<string, { crf: number; audioBitrate: number; videoCodec: string; audioCodec: string }> = {
    visually_lossless: { crf: 15, audioBitrate: 256, videoCodec: 'h264', audioCodec: 'aac' },
    high: { crf: 18, audioBitrate: 192, videoCodec: 'h264', audioCodec: 'aac' },
    medium: { crf: 23, audioBitrate: 128, videoCodec: 'h264', audioCodec: 'aac' },
    low: { crf: 28, audioBitrate: 96, videoCodec: 'h264', audioCodec: 'aac' }
  }
  const preset = presets[val]
  if (preset) {
    videoCodec.value = preset.videoCodec
    audioCodec.value = preset.audioCodec
    videoCRF.value = preset.crf
    audioBitrate.value = preset.audioBitrate
  }
}

const needsReEncode = computed(() =>
  videoCodec.value !== 'copy' || audioCodec.value !== 'copy'
)

watch(encodingConfig, () => { resetOutputStatus() }, { deep: true })

const resetEncodingConfig = () => {
  videoCodec.value = 'copy'
  videoResolution.value = 'original'
  videoFrameRate.value = 'original'
  videoBitrate.value = undefined
  videoCRF.value = 18
  audioCodec.value = 'copy'
  audioBitrate.value = 128
  audioSampleRate.value = 'original'
  outputFormat.value = 'mp4'
  compressionPreset.value = 'visually_lossless'
  logUserAction('编码设置已重置')
}

const ffmpegBaseURL = 'https://unpkg.com/@ffmpeg/core-mt@0.12.10/dist/esm'

const clipDurationSeconds = computed(() => {
  if (slideRange.value.length === 2) {
    return (slideRange.value[1] - slideRange.value[0]).toFixed(2)
  }
})

const formatTime = (seconds: number): string => {
  const m = Math.floor(seconds / 60)
  const s = Math.floor(seconds % 60)
  return `${String(m).padStart(2, '0')}:${String(s).padStart(2, '0')}`
}

const leftPercent = computed(() => {
  if (!videoDuration.value) return '0%'
  return `${(slideRange.value[0] / videoDuration.value) * 100}%`
})

const rightPercent = computed(() => {
  if (!videoDuration.value) return '0%'
  return `${100 - (slideRange.value[1] / videoDuration.value) * 100}%`
})

onMounted(() => {
  dayjs.extend(objectSupport);

  if (!window.WebAssembly) {
    alert("你的浏览器不支持 WebAssembly")
  }
})

onBeforeMount(() => {
  if (rawFileBlobUrl.value) {
    window.URL.revokeObjectURL(rawFileBlobUrl.value)
  }
})

const onSelectMediaFileClickHandler = () => {
  const fileEl = document.createElement('input')
  fileEl.setAttribute('type', 'file')
  fileEl.setAttribute('accept', 'video/*')
  fileEl.addEventListener('change', () => {
    const file = fileEl.files?.[0]
    if (!file) {
      MessagePlugin.error("请选择一个文件")
    }
    isVideoLoading.value = true
    rawFile.value = file;
    rawFileBlobUrl.value = window.URL.createObjectURL(file as File);
    logUserAction(`选择文件: ${file.name} (${prettyBytes(file.size)})`)

    resetOutputStatus()
  })
  fileEl.click()
}

const onVideoLoadedDataHandler = () => {
  videoDuration.value = videoRef.value?.duration ?? -1
  isVideoLoading.value = false
  isVideoLoaded.value = true
  const gap = Math.round(videoDuration.value / 4)
  slideRange.value = [gap, gap * 3]
  logUserAction(`视频加载完成: ${formatTime(videoDuration.value)}`)
}

const resetOutputStatus = () => {
  isFFmpegProcessing.value = false
  isOutputed.value = false
  outputVideoBlob.value = null
  if (outputVideoBlobUrl.value) {
    URL.revokeObjectURL(outputVideoBlobUrl.value)
    outputVideoBlobUrl.value = ''
    videoOutoutRef.value?.setAttribute('src', '')
  }
  outputUsageDuration.value = 0
  generatePercent.value = '0%'
}

const onStartTransCodeClickHandler = async () => {
  resetOutputStatus()
  logUserAction('开始生成')
  try {
    isFFmpegProcessing.value = true;

    const startTime = performance.now();

    await setupFFmpeg()

    const endTime = performance.now()
    outputUsageDuration.value = (endTime - startTime)

    isOutputed.value = true

    logUserAction(`生成完成: ${(outputUsageDuration / 1000).toFixed(2)}s`)
    NotificationPlugin.success({ title: "生成完毕" })
    setTimeout(() => {
      generatePercent.value = '0%'
    }, 1000);
  } catch (error) {
    console.error(error)
    logUserAction(`生成失败: ${error}`)
  } finally {
    isFFmpegProcessing.value = false;
  }
}

const setupFFmpeg = async () => {
  console.info('start load');
  if (!ffmpegRef.value) {
    const ffmpeg = new FFmpeg()
    ffmpegRef.value = ffmpeg

    ffmpeg.on("progress", ({ progress, time }) => {
      console.info(progress, time);
      generatePercent.value = (progress * 100).toFixed(2)
    })

    ffmpeg.on('log', ({ message }) => {
      const ts = new Date().toISOString().slice(11, 23)
      const line = `[${ts}] ${message}`
      console.log(line)
      ffmpegLogs.value.push(line)
    });
    // toBlobURL is used to bypass CORS issue, urls with the same
    // domain can be used directly.
    await ffmpeg.load({
      coreURL: await toBlobURL(`${ffmpegBaseURL}/ffmpeg-core.js`, 'text/javascript'),
      wasmURL: await toBlobURL(`${ffmpegBaseURL}/ffmpeg-core.wasm`, 'application/wasm'),
      workerURL: await toBlobURL(`${ffmpegBaseURL}/ffmpeg-core.worker.js`, 'text/javascript')
    });
  }
  console.info('ffmpeg Loaded!');
  await ffmpegTransCode()
}

const ffmpegTransCode = (): Promise<void> => {
  return new Promise(async (resolve, reject) => {
    try {
      const ffmpeg = ffmpegRef.value

      const inputFileName = rawFile.value?.name ?? 'input.mp4';
      await ffmpeg!.writeFile(inputFileName, await fetchFile(rawFile.value));

      const startTime = dayjs({ second: slideRange.value[0] }).format('HH:mm:ss')
      const endTime = dayjs({ second: slideRange.value[1] }).format('HH:mm:ss')

      const outputFileName = `output.${outputFormat.value}`
      const args: string[] = ['-ss', startTime, '-to', endTime, '-i', inputFileName]

      // 视频编码
      if (videoCodec.value === 'copy') {
        args.push('-c:v', 'copy')
      } else {
        args.push('-c:v', 'libx264', '-crf', String(videoCRF.value))
        if (videoFrameRate.value !== 'original') {
          args.push('-r', videoFrameRate.value)
        }
        if (videoResolution.value !== 'original') {
          args.push('-vf', `scale=-2:${videoResolution.value}`)
        }
        if (videoBitrate.value) {
          args.push('-b:v', `${videoBitrate.value}k`)
        }
      }

      // 音频编码
      if (audioCodec.value === 'copy') {
        args.push('-c:a', 'copy')
      } else {
        args.push('-c:a', audioCodec.value, '-b:a', `${audioBitrate.value}k`)
        if (audioSampleRate.value !== 'original') {
          args.push('-ar', audioSampleRate.value)
        }
      }

      args.push('-movflags', '+faststart', outputFileName)

      const argsStr = args.join(' ')
      console.info('[ffmpeg args]', argsStr)
      logUserAction(`FFmpeg 参数: ${argsStr}`)
      await ffmpeg!.exec(args);
      const data = await ffmpegRef.value!.readFile(outputFileName) as TypedArray;

      const mimeMap: Record<string, string> = { mp4: 'video/mp4', mkv: 'video/x-matroska' }
      outputVideoBlob.value = new Blob([data.buffer], { type: mimeMap[outputFormat.value] || 'video/mp4' });
      outputVideoBlobUrl.value = window.URL.createObjectURL(outputVideoBlob.value)
      console.info(outputVideoBlob.value);

      resolve()
    } catch (error) {
      reject(error)
    }
  })
}

const onDownloadOutputClickHandler = () => {
  if (outputVideoBlobUrl.value) {
    const fileName = `[output]${rawFile.value?.name.replace(/\.[^.]+$/, '')}.${outputFormat.value}`
    logUserAction(`下载: ${fileName}`)
    NotificationPlugin.success({ title: "正在下载", content: "请留意浏览器下载列表" })
    saveAs(outputVideoBlobUrl.value, fileName)
  } else {
    NotificationPlugin.error({ title: "流不存在" })
  }
}

const tempSlideValue = ref<number[]>([])
const onSliderChangeHandler = (value: number[]) => {
  if (tempSlideValue.value[0] !== value[0]) {
    moveVideoTime(value[0])
  } else if (tempSlideValue.value[1] !== value[1]) {
    moveVideoTime(value[1])
  }

  function moveVideoTime(time: number) {
    if (videoRef.value) {
      videoRef.value.pause()
      videoRef.value.currentTime = time
    }
  }
  tempSlideValue.value = value
}
</script>

<style>
/* ===== TDesign 品牌色覆写 — 炭灰 ===== */
:root {
  --td-brand-color: #1F2937;
  --td-brand-color-1: #F3F4F6;
  --td-brand-color-2: #E5E7EB;
  --td-brand-color-3: #D1D5DB;
  --td-brand-color-4: #9CA3AF;
  --td-brand-color-5: #6B7280;
  --td-brand-color-6: #4B5563;
  --td-brand-color-7: #374151;
  --td-brand-color-8: #1F2937;
  --td-brand-color-9: #111827;
  --td-brand-color-10: #0B1016;
  --td-brand-color-hover: #374151;
  --td-brand-color-active: #1F2937;
  --td-brand-color-disabled: #D1D5DB;
  --td-brand-color-focus: rgba(31, 41, 55, 0.12);
  --td-brand-color-light: #F3F4F6;
  --td-brand-color-light-hover: #E5E7EB;
  --td-success-color: #1F2937;
  --td-success-color-hover: #374151;
  --td-success-color-active: #111827;
}

html, body, #app {
  margin: 0;
  padding: 0;
  min-height: 100vh;
  background: #F8F9FB;
  color: #1A1D23;
}
</style>

<style lang="scss" scoped>
/* ===== 变量 ===== */
$bg-page: #F8F9FB;
$bg-surface: #FFFFFF;
$bg-subtle: #F3F4F6;
$border-light: #EAECF0;
$border: #E2E5EA;
$accent: #1F2937;
$text-primary: #1A1D23;
$text-secondary: #5F6B7A;
$text-muted: #98A1AE;
$radius-lg: 16px;
$shadow-card: 0 1px 2px rgba(0,0,0,0.04), 0 2px 8px rgba(0,0,0,0.04);
$shadow-card-hover: 0 1px 2px rgba(0,0,0,0.04), 0 4px 16px rgba(0,0,0,0.06);

/* ===== 全屏布局 ===== */
.app {
  height: 100vh;
  min-width: 1024px;
  display: flex;
  flex-direction: column;
  background: $bg-page;
}

/* ===== 顶栏 ===== */
.app-header {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 14px 24px;
  background: $bg-surface;
  border-bottom: 1px solid $border-light;
}

.header-brand {
  display: flex;
  align-items: center;
  gap: 10px;
}

.brand-icon {
  width: 24px;
  height: 24px;
  color: $accent;
}

.brand-title {
  font-size: 20px;
  font-weight: 700;
  color: $text-primary;
  margin: 0;
  letter-spacing: -0.5px;
}

.header-tag {
  font-size: 11px;
  font-weight: 500;
  color: $text-muted;
  background: $bg-subtle;
  padding: 3px 10px;
  border-radius: 100px;
  border: 1px solid $border-light;
}

/* ===== 视频区 (撑满剩余高度) ===== */
.video-grid {
  flex: 1;
  min-height: 0;
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
}

.video-card {
  background: $bg-surface;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  box-shadow: $shadow-card;
  transition: border-color 0.25s, box-shadow 0.25s;
  min-height: 0;

  &:hover {
    box-shadow: $shadow-card-hover;
  }

  &.output {
    position: relative;

    &.active {
      border-color: $accent;
    }
  }
}

.card-top {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px 18px;
  background: $bg-surface;
  border-bottom: 1px solid $border-light;
}

.card-badge {
  font-size: 12px;
  font-weight: 600;
  padding: 2px 10px;

  &.preview-badge {
    background: $bg-subtle;
    color: $text-secondary;
  }

  &.output-badge {
    background: $bg-subtle;
    color: $accent;
  }
}

.processing-hint {
  font-size: 12px;
  font-weight: 500;
  color: #E07B2C;
  animation: pulse 1.2s ease-in-out infinite;
}

.done-hint {
  font-size: 12px;
  font-weight: 500;
  color: $accent;
}

@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.4; }
}

.card-player {
  flex: 1;
  min-height: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #1A1D23;

  :deep(.t-loading__text) {
    color: #9CA3AF;
  }

  video {
    width: 100%;
    height: 100%;
    object-fit: contain;
    outline: none;
  }
}

.progress-track {
  flex-shrink: 0;
  height: 3px;
  background: $border-light;

  .progress-fill {
    height: 100%;
    width: var(--output-percent);
    background: $accent;
    transition: width 0.15s linear;
  }
}

/* ===== 底部操作栏 ===== */
.control-bar {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  gap: 18px;
  padding: 10px 24px;
  background: $bg-surface;
  border-top: 1px solid $border-light;
  box-shadow: 0 -1px 3px rgba(0,0,0,0.03);
  overflow-x: auto;
  white-space: nowrap;
}

.bar-section {
  display: flex;
  align-items: center;
  gap: 8px;
  flex-shrink: 0;
}

.bar-file {
  min-width: 0;
}

.bar-timeline {
  flex: 1;
  min-width: 320px;
  gap: 0;
}

/* 时间轴节点 (开始/结束标签) */
.tl-node {
  flex-shrink: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1px;
  min-width: 48px;
}

.tl-time {
  font-size: 14px;
  font-weight: 600;
  font-variant-numeric: tabular-nums;
  font-family: 'SF Mono', 'Cascadia Code', monospace;
  color: $text-primary;
}

.tl-tag {
  font-size: 10px;
  font-weight: 500;
  color: $text-muted;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

/* 时间轴滑块轨道 */
.tl-track {
  flex: 1;
  min-width: 120px;
  padding: 0 8px;
  position: relative;
}

/* 选中范围可视化高亮 */
.tl-range-visual {
  position: absolute;
  top: 50%;
  left: 0;
  right: 0;
  height: 4px;
  transform: translateY(-50%);
  border-radius: 2px;
  background: $border-light;
  pointer-events: none;

  &::after {
    content: '';
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: $accent;
    border-radius: 2px;
    opacity: 0.15;
  }
}

/* 时长芯片 */
.tl-duration {
  flex-shrink: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1px;
  padding: 4px 14px;
  background: $bg-subtle;
  border-radius: 100px;
  min-width: 52px;
}

.tl-duration-value {
  font-size: 15px;
  font-weight: 700;
  font-variant-numeric: tabular-nums;
  color: $accent;
  font-family: 'SF Mono', 'Cascadia Code', monospace;
}

.tl-duration-label {
  font-size: 10px;
  font-weight: 500;
  color: $text-muted;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.bar-actions {
  gap: 10px;
}

.btn-icon {
  font-weight: 600;
  font-size: 14px;
  line-height: 1;
}

.bar-meta {
  font-size: 12px;
  color: $text-secondary;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 180px;

  &.mono {
    font-family: 'SF Mono', 'Cascadia Code', monospace;
    font-size: 11px;
    color: $text-muted;
  }

  &.accent {
    color: $accent;
    font-weight: 600;
  }
}

.bar-sep {
  color: #D1D5DB;
  flex-shrink: 0;
  font-size: 13px;

  &.pipe {
    color: $border;
  }
}

.bar-meta-group {
  display: flex;
  align-items: center;
  gap: 4px;
  flex-shrink: 0;
}

.bar-meta-label {
  font-size: 10px;
  font-weight: 500;
  color: $text-muted;
  text-transform: uppercase;
  letter-spacing: 0.4px;
}

/* ===== 日志抽屉 ===== */
.log-drawer-header {
  display: flex;
  align-items: center;
  gap: 10px;
}

.log-count {
  font-size: 12px;
  color: $text-muted;
  font-weight: 400;
}
.log-container {
  font-family: 'SF Mono', 'Cascadia Code', 'Consolas', monospace;
  font-size: 12px;
  line-height: 1.7;
}

.log-empty {
  color: $text-muted;
  text-align: center;
  padding: 40px 0;
  font-family: inherit;
}

.log-line {
  padding: 1px 0;
  color: $text-secondary;
  word-break: break-all;
  border-bottom: 1px solid $border-light;

  &:last-child {
    border-bottom: none;
  }
}

/* ===== TDesign 微调 ===== */
:deep(.t-slider__track) {
  background: $border-light;
}

:deep(.t-slider__rail) {
  background: $border-light;
}

:deep(.t-button--variant-outline) {
  border-color: $border;
  color: $text-primary;

  &:hover {
    border-color: $accent;
    color: $accent;
  }
}

/* ===== 编码设置面板 ===== */
.settings-section {
  flex-shrink: 0;
  background: $bg-surface;
  border-top: 1px solid $border-light;
  overflow: hidden;
  transition: max-height 0.3s ease;

  &.expanded {
    border-bottom: 1px solid $border-light;
  }
}

.settings-toggle {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 7px 24px;
  cursor: pointer;
  user-select: none;
  font-size: 12px;
  font-weight: 500;
  color: $text-secondary;
  transition: background 0.15s;

  &:hover {
    background: $bg-subtle;
  }

  svg {
    transition: transform 0.25s ease;
    flex-shrink: 0;

    &.rotated {
      transform: rotate(-180deg);
    }
  }
}

.settings-hint {
  font-size: 10px;
  font-weight: 500;
  color: $text-muted;
  background: $bg-subtle;
  padding: 1px 8px;
  border-radius: 100px;
  margin-left: 4px;

  &.re-encode {
    color: #E07B2C;
    background: #FFF7ED;
  }
}

.settings-body {
  padding: 10px 24px 14px;
  border-top: 1px solid $border-light;
  background: $bg-subtle;
}

.settings-row {
  display: flex;
  gap: 24px;
}

.settings-group {
  flex: 1;
  min-width: 0;
}

.settings-group-title {
  font-size: 11px;
  font-weight: 600;
  color: $text-muted;
  text-transform: uppercase;
  letter-spacing: 0.6px;
  margin-bottom: 8px;
  padding-bottom: 4px;
  border-bottom: 1px solid $border-light;
}

.settings-field {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 6px;

  .field-label {
    flex-shrink: 0;
    width: 64px;
    font-size: 11px;
    font-weight: 500;
    color: $text-secondary;
    text-align: right;
  }

  .field-hint {
    flex-shrink: 0;
    font-size: 11px;
    color: $text-muted;
    font-variant-numeric: tabular-nums;
    min-width: 80px;
  }

  :deep(.t-slider) {
    flex: 1;
  }
}
</style>