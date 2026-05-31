<template>
  <div class="rdp-page">
    <div class="rdp-toolbar">
      <span class="connection-status" :style="{ color: statusColor }">
        {{ statusText }}
      </span>
      <span v-if="hostIp" class="host-info">{{ hostIp }}</span>
      <div class="toolbar-right">
        <el-button size="small" :title="isFullscreen ? '退出全屏' : '全屏'" @click="toggleFullscreen">
          <el-icon><FullScreen v-if="!isFullscreen" /><Close v-else /></el-icon>
        </el-button>
      </div>
    </div>
    <div ref="displayContainer" class="rdp-display" />
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount } from 'vue'
import { useRoute } from 'vue-router'
import { ElMessage } from 'element-plus'
import { FullScreen, Close } from '@element-plus/icons-vue'
import { useFullscreen } from '../composables/useFullscreen'
import { useConnectionStatus } from '../composables/useConnectionStatus'
import Guacamole from 'guacamole-common-js'

const route = useRoute()
const key = route.query.key as string
const hostIp = route.query.host as string

const displayContainer = ref<HTMLDivElement>()

const status = ref<'connecting' | 'connected' | 'disconnected' | 'error'>('connecting')
const error = ref('')

const { statusColor, statusText } = useConnectionStatus(status, error)
const { isFullscreen, toggleFullscreen } = useFullscreen(onFullscreenChange)

let guacClient: any = null
let tunnel: Guacamole.WebSocketTunnel | null = null
let resizeObserver: ResizeObserver | null = null

function getToolbarHeight(): number {
  const toolbarEl = document.querySelector('.rdp-toolbar') as HTMLElement
  return toolbarEl ? toolbarEl.offsetHeight : 0
}

function updateContainerSize() {
  if (!displayContainer.value) return
  const toolbarH = isFullscreen.value ? 0 : getToolbarHeight()
  displayContainer.value.style.top = toolbarH + 'px'
  displayContainer.value.style.width = '100vw'
  displayContainer.value.style.height = `calc(100vh - ${toolbarH}px)`
}

function fitDisplay() {
  if (!guacClient || !displayContainer.value) return
  const display = guacClient.getDisplay()
  const toolbarH = isFullscreen.value ? 0 : getToolbarHeight()
  // 使用 offsetWidth/offsetHeight 获取容器实际尺寸
  const container = displayContainer.value
  const containerW = container.offsetWidth
  const containerH = container.offsetHeight
  const displayW = display.getWidth()
  const displayH = display.getHeight()
  if (displayW === 0 || displayH === 0 || containerW === 0 || containerH === 0) return
  
  // 使用 Math.max 填满容器，确保没有空白区域
  // 远程桌面可能会有部分内容被裁剪，但可以通过全屏查看完整内容
  const scale = Math.max(containerW / displayW, containerH / displayH)
  display.scale(scale)
}

function onFullscreenChange() {
  const toolbarEl = document.querySelector('.rdp-toolbar') as HTMLElement
  if (toolbarEl) {
    toolbarEl.style.display = isFullscreen.value ? 'none' : 'flex'
  }
  requestAnimationFrame(() => {
    requestAnimationFrame(() => {
      updateContainerSize()
      fitDisplay()
      if (guacClient && status.value === 'connected') {
        guacClient.sendSize(window.innerWidth, window.innerHeight)
      }
      setTimeout(() => {
        updateContainerSize()
        fitDisplay()
      }, 300)
    })
  })
}

onMounted(() => {
  if (!key) {
    ElMessage.error('缺少连接密钥')
    return
  }

  const pageEl = document.querySelector('.rdp-page') as HTMLElement
  if (pageEl) {
    pageEl.style.position = 'relative'
    pageEl.style.height = '100vh'
    pageEl.style.overflow = 'hidden'
  }

  const backendHost = import.meta.env.VITE_API_HOST || 'localhost:8000'
  const protocol = location.protocol === 'https:' ? 'wss:' : 'ws:'
  const toolbarH = getToolbarHeight()
  const wsUrl = `${protocol}//${backendHost}/ws/v1/rdp/${key}?width=${window.innerWidth}&height=${window.innerHeight - toolbarH}`

  tunnel = new Guacamole.WebSocketTunnel(wsUrl)
  guacClient = new Guacamole.Client(tunnel)

  const displayEl = guacClient.getDisplay().getElement()
  displayContainer.value!.appendChild(displayEl)
  displayEl.style.cursor = 'none'

  const container = displayContainer.value!
  container.style.position = 'fixed'
  container.style.left = '0'
  container.style.overflow = 'hidden'
  container.style.zIndex = '1'
  updateContainerSize()

  const guacDisplay = guacClient.getDisplay()

  const mouse = new Guacamole.Mouse(displayEl)
  mouse.onmousedown = mouse.onmouseup = mouse.onmousemove = (mouseState: any) => {
    guacClient.sendMouseState(mouseState, true)
  }

  const keyboard = new Guacamole.Keyboard(document)
  keyboard.onkeydown = (keysym: number) => {
    guacClient.sendKeyEvent(1, keysym)
  }
  keyboard.onkeyup = (keysym: number) => {
    guacClient.sendKeyEvent(0, keysym)
  }

  tunnel.onerror = (errorMsg: any) => {
    console.error('[RDP] Tunnel error:', errorMsg)
    status.value = 'error'
    error.value = (errorMsg && errorMsg.message) || '连接失败'
    ElMessage.error(error.value)
  }

  tunnel.onstatechange = (tunnelState: number) => {
    switch (tunnelState) {
      case Guacamole.Tunnel.State.OPEN:
        status.value = 'connected'
        break
      case Guacamole.Tunnel.State.CLOSED:
        status.value = 'disconnected'
        break
      case Guacamole.Tunnel.State.CONNECTING:
        status.value = 'connecting'
        break
    }
  }

  guacClient.connect('')

  guacDisplay.onresize = () => {
    fitDisplay()
  }

  let synced = false
  guacClient.onsync = () => {
    if (!synced) {
      synced = true
      fitDisplay()
    }
  }

  // 使用 ResizeObserver 监听 .rdp-page 尺寸变化
  if (pageEl) {
    resizeObserver = new ResizeObserver(() => {
      updateContainerSize()
      requestAnimationFrame(() => fitDisplay())
      if (guacClient && status.value === 'connected') {
        const h = isFullscreen.value ? 0 : getToolbarHeight()
        guacClient.sendSize(window.innerWidth, window.innerHeight - h)
      }
    })
    resizeObserver.observe(pageEl)
  }

  // window.resize 作为备用监听（某些浏览器最大化可能不触发 ResizeObserver）
  let resizeTimer: ReturnType<typeof setTimeout> | null = null
  window.addEventListener('resize', () => {
    if (resizeTimer) clearTimeout(resizeTimer)
    resizeTimer = setTimeout(() => {
      updateContainerSize()
      requestAnimationFrame(() => fitDisplay())
      if (guacClient && status.value === 'connected') {
        const h = isFullscreen.value ? 0 : getToolbarHeight()
        guacClient.sendSize(window.innerWidth, window.innerHeight - h)
      }
    }, 100)
  })
})

onBeforeUnmount(() => {
  resizeObserver?.disconnect()
  tunnel?.disconnect()
  guacClient = null
  tunnel = null
})
</script>

<style scoped>
.rdp-page {
  position: relative;
  height: 100vh;
  background: #1e1e1e;
  overflow: hidden;
}

.rdp-toolbar {
  position: relative;
  z-index: 10;
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 6px 14px;
  background: #2d2d2d;
  border-bottom: 1px solid #3d3d3d;
}

.connection-status {
  font-size: 13px;
}

.host-info {
  font-size: 11px;
  color: #909399;
}

.toolbar-right {
  display: flex;
  gap: 6px;
  margin-left: auto;
  align-items: center;
}

.rdp-display {
  overflow: hidden;
}
</style>
