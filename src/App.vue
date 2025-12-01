<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'
import { ElMessage, ElNotification, ElLoading } from 'element-plus'
import { Microphone, Moon, Sunny, Search, Link, VideoPlay, InfoFilled, Setting, User, Download, Document, Check, FolderOpened, Loading, List, Refresh, ElementPlus } from '@element-plus/icons-vue'
// 使用懒加载导入组件以减少初始包大小
import MusicPlayer from './components/MusicPlayer.vue'
import PlaylistDetail from './components/PlaylistDetail.vue'
import musicApi, { QUALITY_LEVELS } from './services/musicApi.js'
import { setCookie, getCookie } from './utils/cookies.js'

// 导入新的模块化功能
import { isDark, toggleTheme, initThemeFromLocalStorage } from './utils/themeManager.js'
import { settings, loadSettings, saveSettings } from './utils/settingsManager.js'
import { 
  musicUrl, loading, musicInfo, playlistUrl, playlistLoading, playlistInfo, albumUrl, albumLoading, albumInfo,elapsedTime, parseMusic, parsePlaylist, parseAlbum, clearMusicResult, clearPlaylistResult, clearAlbumResult, setExampleUrl, cleanupTimer,
  currentParsingTrack, parsingProgress
} from './utils/parseManager.js'
import { 
  currentPage, pageSize, totalTracks, displayTracks, 
  updateDisplayTracks, handlePageChange
} from './utils/paginationManager.js'
import { isMobile, initDeviceDetection, cleanupDeviceDetection } from './utils/deviceDetector.js'
import { getCurrentExampleLinks } from './utils/exampleData.js'
 

// 本地组件状态
const selectedQuality = ref('lossless') // 默认无损音质
const showSettingsDialog = ref(false)
const showWelcomeDialog = ref(false)
 
 
// 已移除极验弹窗相关状态
// 主题切换全屏Loading
const toggleThemeWithLoading = () => {
  // 根据当前主题选择不同的遮罩背景
  const bg = isDark.value ? 'rgba(0,0,0,0.6)' : 'rgba(255,255,255,0.7)'
  const text = isDark.value ? '切换为浅色模式…' : '切换为深色模式…'
  const loading = ElLoading.service({
    fullscreen: true,
    lock: true,
    text,
    background: bg
  })
  // 让遮罩先渲染一帧，再切换主题
  requestAnimationFrame(() => {
    toggleTheme()
    // themeManager 的动画约600ms，这里略微延长后关闭遮罩
    setTimeout(() => {
      loading.close()
    }, 700)
  })
}

// 视图切换相关
const currentView = ref('music', 'playlist', 'album') // 'music', 'playlist', or 'album'

// 音质选项
const qualityOptions = Object.entries(QUALITY_LEVELS).map(([value, label]) => ({
  value,
  label
}))

 

// 环境信息（是否本地开发）
const isLocalEnv = computed(() => {
  if (typeof window === 'undefined') return false
  const host = window.location.hostname
  return host === 'localhost' || host === '127.0.0.1' || host === '::1'
})

 


// 组件卸载时清理监听器
onUnmounted(() => {
  cleanupDeviceDetection()
  cleanupTimer()
})

// 初始化主题和设置
onMounted(() => {
  loadSettings()
  
  // 初始化设备检测
  initDeviceDetection()
  
  // 初始化主题（仅使用本地存储，不跟随系统）
  initThemeFromLocalStorage()
  
  // 检查是否需要显示欢迎弹窗
  setTimeout(() => {
    const hasSeenWelcome = getCookie('hasSeenWelcome')
    if (!hasSeenWelcome) {
      showWelcomeDialog.value = true
    }
  }, 1000)
})

// 切换视图
const switchView = (view) => {
  currentView.value = view
  // 切换视图时清空对应的结果
  if (view === 'music') {
    clearPlaylistResult()
    clearAlbumResult()
  } else if (view === 'playlist') {
    clearMusicResult()
    clearAlbumResult()
  } else if (view === 'album') {
    clearMusicResult()
    clearPlaylistResult()
  }
}

// 处理欢迎弹窗确认
const handleWelcomeConfirm = () => {
  // 设置cookie，30天过期
  setCookie('hasSeenWelcome', 'true', 30)
  showWelcomeDialog.value = false
}

const useExampleLink = (link, name) => {
  if (currentView.value === 'playlist') {
    setExampleUrl(link, 'playlist')
    ElMessage.success(`已选择示例歌单: ${name}`)
  } else if (currentView.value === 'album') {
    setExampleUrl(link, 'album')
    ElMessage.success(`已选择示例专辑: ${name}`)
  } else {
    setExampleUrl(link, 'music')
    ElMessage.success(`已选择示例歌曲: ${name}`)
  }
}

// 主题模式设置移除：改用悬浮按钮手动切换

// 处理歌曲解析事件
const handleTrackParsed = async (data) => {
  try {
    // 兼容新旧参数格式
    const track = data.track || data
    const quality = data.quality || selectedQuality.value
    
    // 确保quality是字符串类型，避免事件对象污染
    const qualityValue = typeof quality === 'string' ? quality : 'lossless'
    
    // 构造音乐URL用于解析
    const musicUrl = `https://music.163.com/song?id=${track.id}`
    
    // 调用现有的解析逻辑，使用传入的音质参数
    const result = await musicApi.parseMusicInfo(musicUrl, qualityValue)
    
    // 更新音乐信息
    musicInfo.value = result
    
    ElNotification({
      title: '解析成功',
      message: `成功解析歌曲：${result.name} (${result.qualityName})`,
      type: 'success'
    })
  } catch (error) {
    ElMessage.error(error.message || '解析失败，请稍后重试')
  }
}

// 处理歌曲选择事件
const handleTrackSelected = () => {}

const computedPlaylistInfo = computed(() => {
  if (currentView.value === 'playlist' && playlistInfo.value) {
    return playlistInfo.value;
  }
  if (currentView.value === 'album' && albumInfo.value) {
    // Adapt albumInfo to the structure expected by PlaylistDetail
    return {
      name: albumInfo.value.name,
      creator: albumInfo.value.artist,
      time: albumInfo.value.publishTime,
      totalCount: albumInfo.value.trackCount,
      tracks: albumInfo.value.tracks,
      picUrl: albumInfo.value.picUrl,
      isAlbum: true,
    };
  }
  return null;
});


</script>

<template>
  <el-config-provider size="default">
    <el-container class="app-container" direction="vertical">
    <!-- 顶部导航栏 -->
    <el-header class="app-header">
      <div class="header-content">
        <div class="header-left">
          <div class="logo">
            <el-icon class="logo-icon" :size="24">
              <Microphone />
            </el-icon>
            <span class="logo-text">网易云音乐无损解析</span>
          </div>
          <span class="header-view-title">
            {{ currentView === 'music' ? '单曲解析' : (currentView === 'playlist' ? '歌单解析' : '专辑解析') }}
          </span>
        </div>
        
        <div class="header-right">
          <el-button 
            @click="showSettingsDialog = true" 
            type="text" 
            class="settings-btn"
            :icon="Setting"
            circle
            size="large"
          />
        </div>
      </div>
    </el-header>

    <!-- 主要内容区域 -->
    <el-container class="main-container">
      <!-- 主内容与侧栏采用标准 Container 布局 -->
      <el-main class="app-main">
        <!-- 页面标题 -->
        <el-card class="page-header-card" shadow="hover">
          <div class="page-header">
            <h1 class="page-title">网易云音乐无损解析</h1>
            <p class="page-description">永久免费的网易云音乐高品质解析</p>
          </div>
        </el-card>

        <!-- 功能切换卡片 -->
        <el-card class="view-switcher-container" shadow="hover">
          <template #header>
            <div class="card-header">
              <el-icon><Refresh /></el-icon>
              <span>选择解析方式</span>
            </div>
          </template>
          <div class="view-buttons">
            <el-button 
              class="view-btn"
              :type="currentView === 'music' ? 'primary' : 'default'"
              :plain="currentView !== 'music'"
              round
              size="large"
              @click="switchView('music')"
            >单曲解析</el-button>

            <el-button 
              class="view-btn"
              :type="currentView === 'playlist' ? 'primary' : 'default'"
              :plain="currentView !== 'playlist'"
              round
              size="large"
              @click="switchView('playlist')"
            >歌单解析</el-button>

            <el-button 
              class="view-btn"
              :type="currentView === 'album' ? 'primary' : 'default'"
              :plain="currentView !== 'album'"
              round
              size="large"
              @click="switchView('album')"
            >专辑解析</el-button>
          </div>
        </el-card>

        <!-- 单曲解析视图 -->
        <div class="view-container" v-show="currentView === 'music'">
          <el-card class="input-card" shadow="hover">
            <template #header>
              <div class="card-header">
                <el-icon><Link /></el-icon>
                <span>音乐链接解析</span>
              </div>
            </template>
            <el-form :model="{ musicUrl, selectedQuality }"  label-position="left">
              <el-form-item label="音乐链接">
                <el-input
                  v-model="musicUrl"
                  placeholder="请输入网易云音乐分享链接"
                  clearable
                  @keyup.enter="parseMusic"
                >
                  <template #prepend>
                    <el-icon><Link /></el-icon>
                  </template>
                </el-input>
              </el-form-item>

              <div class="form-items-row">
                <el-form-item label="音质选择" class="quality-item">
                  <el-select v-model="selectedQuality" placeholder="请选择音质" style="width: 200px">
                    <el-option
                      v-for="option in qualityOptions"
                      :key="option.value"
                      :label="option.label"
                      :value="option.value"
                    >
                      <div style="display: flex; align-items: center; justify-content: space-between;">
                        <span>{{ option.label }}</span>
                        <el-tag v-if="option.value === 'jymaster'" type="danger" size="small">最高</el-tag>
                        <el-tag v-else-if="option.value === 'lossless'" type="success" size="small">推荐</el-tag>
                      </div>
                    </el-option>
                  </el-select>
                </el-form-item>
                
              </div>

              <el-form-item>
                <el-button 
                  type="primary" 
                  @click="() => parseMusic(selectedQuality)" 
                  :loading="loading"
                  :disabled="!musicUrl.trim()"
                  :icon="Search"
                  style="width: 100%"
                >
                  {{ loading ? '解析中...' : '开始解析' }}
                </el-button>
              </el-form-item>
            </el-form>
          </el-card>
        </div>

        <!-- 歌单解析视图 -->
        <div class="view-container" v-show="currentView === 'playlist'">
          <el-card class="input-card" shadow="hover">
            <template #header>
              <div class="card-header">
                <el-icon><Document /></el-icon>
                <span>歌单链接解析</span>
              </div>
            </template>
            <el-form :model="{ playlistUrl }"  label-position="left">
              <el-form-item label="歌单链接">
                <el-input
                  v-model="playlistUrl"
                  placeholder="请输入网易云音乐歌单链接（支持短链接 163cn.tv/xxx）"
                  clearable
                  @keyup.enter="parsePlaylist"
                >
                  <template #prepend>
                    <el-icon><Link /></el-icon>
                  </template>
                </el-input>
              </el-form-item>

              <div class="form-items-row">
                <el-form-item label="音质选择" class="quality-item">
                  <el-select v-model="selectedQuality" placeholder="请选择音质" style="width: 200px">
                    <el-option
                      v-for="option in qualityOptions"
                      :key="option.value"
                      :label="option.label"
                      :value="option.value"
                    >
                      <div style="display: flex; align-items: center; justify-content: space-between;">
                        <span>{{ option.label }}</span>
                        <el-tag v-if="option.value === 'jymaster'" type="danger" size="small">最高</el-tag>
                        <el-tag v-else-if="option.value === 'lossless'" type="success" size="small">推荐</el-tag>
                      </div>
                    </el-option>
                  </el-select>
                </el-form-item>
                
              </div>

              <el-form-item>
                <el-button 
                  type="primary" 
                  @click="() => parsePlaylist(updateDisplayTracks)" 
                  :loading="playlistLoading"
                  :disabled="!playlistUrl.trim()"
                  :icon="Search"
                  style="width: 100%"
                >
                  {{ playlistLoading ? '解析中...' : '开始解析' }}
                </el-button>
              </el-form-item>
            </el-form>
          </el-card>
        </div>

        <!-- 专辑解析视图 -->
        <div class="view-container" v-show="currentView === 'album'">
          <el-card class="input-card" shadow="hover">
            <template #header>
              <div class="card-header">
                <el-icon><FolderOpened /></el-icon>
                <span>专辑链接解析</span>
              </div>
            </template>
            <el-form :model="{ albumUrl }"  label-position="left">
              <el-form-item label="专辑链接">
                <el-input
                  v-model="albumUrl"
                  placeholder="请输入网易云音乐专辑链接（支持短链接 163cn.tv/xxx）"
                  clearable
                  @keyup.enter="parseAlbum"
                >
                  <template #prepend>
                    <el-icon><Link /></el-icon>
                  </template>
                </el-input>
              </el-form-item>

              <div class="form-items-row">
                <el-form-item label="音质选择" class="quality-item">
                  <el-select v-model="selectedQuality" placeholder="请选择音质" style="width: 200px">
                    <el-option
                      v-for="option in qualityOptions"
                      :key="option.value"
                      :label="option.label"
                      :value="option.value"
                    >
                      <div style="display: flex; align-items: center; justify-content: space-between;">
                        <span>{{ option.label }}</span>
                        <el-tag v-if="option.value === 'jymaster'" type="danger" size="small">最高</el-tag>
                        <el-tag v-else-if="option.value === 'lossless'" type="success" size="small">推荐</el-tag>
                      </div>
                    </el-option>
                  </el-select>
                </el-form-item>
                
              </div>

              <el-form-item>
                <el-button 
                  type="primary" 
                  @click="() => parseAlbum(updateDisplayTracks)" 
                  :loading="albumLoading"
                  :disabled="!albumUrl.trim()"
                  :icon="Search"
                  style="width: 100%"
                >
                  {{ albumLoading ? '解析中...' : '开始解析' }}
                </el-button>
              </el-form-item>
            </el-form>
          </el-card>
        </div>

        <!-- 歌单/专辑详情组件 -->
        <div v-if="computedPlaylistInfo" class="playlist-section">
          <PlaylistDetail 
            :playlist-info="computedPlaylistInfo" 
            :display-tracks="displayTracks"
            :current-page="currentPage"
            :page-size="pageSize"
            :total-tracks="totalTracks"
            :selected-quality="selectedQuality"
            :settings="settings"
            @track-parsed="handleTrackParsed"
            @track-selected="handleTrackSelected"
            @page-change="handlePageChange"
          />
        </div>

        <!-- 播放器区域 -->
        <div v-if="musicInfo" class="player-section">
          <Suspense>
            <template #default>
              <MusicPlayer :music-info="musicInfo" :settings="settings" />
            </template>
            <template #fallback>
              <div class="loading-placeholder">
                <el-skeleton :rows="3" animated />
                <div class="loading-text">正在加载播放器...</div>
              </div>
            </template>
          </Suspense>
        </div>

        <!-- 仅在移动端或单栏模式下显示合并后的侧栏卡片 -->
        <div class="side-cards-inline" v-if="isMobile || settings.layoutMode !== 'dual-column'">
          <div class="help-section">
            <el-card shadow="hover">
              <template #header>
                <div class="card-header">
                  <el-icon><VideoPlay /></el-icon>
                  <span>示例歌曲与支持格式</span>
                </div>
              </template>
              <div class="examples-section">
                <h4 class="section-title">{{ currentView === 'playlist' ? '📋 热门歌单' : (currentView === 'album' ? '💿 热门专辑' : '🎧 热门示例') }}</h4>
                <div class="example-list">
                  <div 
                    v-for="(example, index) in getCurrentExampleLinks(currentView)" 
                    :key="index"
                    class="example-item"
                    @click="useExampleLink(example.url, example.name)"
                  >
                    <div class="example-info">
                      <div class="example-name">{{ example.name }}</div>
                      <div class="example-artist">{{ currentView === 'playlist' ? example.creator : example.artist }}</div>
                    </div>
                    <el-button type="text" size="small">使用</el-button>
                  </div>
                </div>
              </div>
              <el-divider />
              <div class="formats-section">
                <h4 class="section-title">📋 支持格式</h4>
                <div class="format-list" v-if="currentView === 'music'">
                  <el-tag class="format-item" type="info">music.163.com/song?id=xxx</el-tag>
                  <el-tag class="format-item" type="info">y.music.163.com/m/song/xxx</el-tag>
                  <el-tag class="format-item" type="info">music.163.com/#/song?id=xxx</el-tag>
                  <el-tag class="format-item" type="info">163cn.tv/xxx</el-tag>
                </div>
                <div class="format-list" v-else-if="currentView === 'playlist'">
                  <el-tag class="format-item" type="success">music.163.com/playlist?id=xxx</el-tag>
                  <el-tag class="format-item" type="success">music.163.com/#/playlist?id=xxx</el-tag>
                  <el-tag class="format-item" type="success">y.music.163.com/m/playlist/xxx</el-tag>
                  <el-tag class="format-item" type="success">163cn.tv/xxx</el-tag>                  
                </div>
                <div class="format-list" v-else-if="currentView === 'album'">
                  <el-tag class="format-item" type="warning">music.163.com/album?id=xxx</el-tag>
                  <el-tag class="format-item" type="warning">music.163.com/album/xxx</el-tag>
                  <el-tag class="format-item" type="warning">music.163.com/#/album?id=xxx</el-tag>
                  <el-tag class="format-item" type="warning">music.163.com/#/album/xxx</el-tag>
                  <el-tag class="format-item" type="warning">y.music.163.com/m/album/xxx</el-tag>
                  <el-tag class="format-item" type="warning">163cn.tv/xxx</el-tag>
                </div>
              </div>
            </el-card>
          </div>

          <el-card class="author-card" shadow="hover">
            <template #header>
              <div class="card-header">
                <el-icon><User /></el-icon>
                <span>作者与联系</span>
              </div>
            </template>
            <div class="author-info">
              <p><strong>作者：</strong> 苏晓晴</p>
              <p><strong>个人博客：</strong> <el-link type="primary" href="https://www.toubiec.cn" target="_blank">点击跳转</el-link></p>
              <p><strong>联系方式：</strong> QQ3074193836</p>
              <p><strong>版权声明：</strong> 本工具仅用于学习交流，不提供任何音频文件存储服务。</p>
              <p><strong>反馈地址：</strong> <el-link type="primary" href="https://github.com/Suxiaoqinx/Netease_url/issues" target="_blank">点击跳转</el-link></p>
              <p>如果您在使用中遇到任何问题，或有任何建议，欢迎联系！</p>
            </div>
          </el-card>
        </div>

        <!-- 全屏歌单解析Loading -->        
        <div v-if="playlistLoading" class="fullscreen-loading">
          <div class="loading-content">
            <el-icon class="loading-icon" :size="48">
              <Loading />
            </el-icon>
            <div class="loading-text">正在解析歌单中...</div>
            <div class="loading-subtitle" v-if="currentParsingTrack">
              正在解析：{{ currentParsingTrack.name }} - {{ currentParsingTrack.artist }}
            </div>
            <div class="loading-subtitle" v-else>
              请稍候，正在获取歌单信息
            </div>
            <div class="loading-progress" v-if="parsingProgress.totalTracks > 0">
              进度：{{ parsingProgress.currentIndex + 1 }} / {{ parsingProgress.totalTracks }}
              (成功：{{ parsingProgress.successfulTracks.filter(t => t.parsed).length }})
            </div>
            <div class="loading-timer">已用时间：{{ elapsedTime }}秒</div>
          </div>
        </div>

        <!-- 免责声明 - 常驻显示 -->
        <el-card class="disclaimer-card" shadow="hover">
          <el-alert
            title="免责声明"
            description="本工具仅供学习交流使用，请支持正版音乐。使用本工具时请遵守相关法律法规，尊重音乐人的劳动成果。"
            type="warning"
            :closable="false"
            show-icon
            center
          />
        </el-card>
      </el-main>

      <!-- 桌面双栏侧栏：在PC端且设置为双栏时显示 -->
      <el-aside
        v-if="!isMobile && settings.layoutMode === 'dual-column'"
        class="app-aside"
        width="340px"
      >
        <div class="side-column">
          <div class="help-section">
            <el-card shadow="hover">
              <template #header>
                <div class="card-header">
                  <el-icon><VideoPlay /></el-icon>
                  <span>示例歌曲与支持格式</span>
                </div>
              </template>
              <div class="examples-section">
                <h4 class="section-title">{{ currentView === 'playlist' ? '📋 热门歌单' : (currentView === 'album' ? '💿 热门专辑' : '🎧 热门示例') }}</h4>
                <div class="example-list">
                  <div 
                    v-for="(example, index) in getCurrentExampleLinks(currentView)" 
                    :key="index"
                    class="example-item"
                    @click="useExampleLink(example.url, example.name)"
                  >
                    <div class="example-info">
                      <div class="example-name">{{ example.name }}</div>
                      <div class="example-artist">{{ currentView === 'playlist' ? example.creator : example.artist }}</div>
                    </div>
                    <el-button type="text" size="small">使用</el-button>
                  </div>
                </div>
              </div>
              <el-divider />
              <div class="formats-section">
                <h4 class="section-title">📋 支持格式</h4>
                <div class="format-list" v-if="currentView === 'music'">
                  <el-tag class="format-item" type="info">music.163.com/song?id=xxx</el-tag>
                  <el-tag class="format-item" type="info">y.music.163.com/m/song/xxx</el-tag>
                  <el-tag class="format-item" type="info">music.163.com/#/song?id=xxx</el-tag>
                  <el-tag class="format-item" type="info">163cn.tv/xxx</el-tag>
                </div>
                <div class="format-list" v-else-if="currentView === 'playlist'">
                  <el-tag class="format-item" type="success">music.163.com/playlist?id=xxx</el-tag>
                  <el-tag class="format-item" type="success">music.163.com/#/playlist?id=xxx</el-tag>
                  <el-tag class="format-item" type="success">y.music.163.com/m/playlist/xxx</el-tag>
                  <el-tag class="format-item" type="success">music.163.com/discover/toplist?id=xxx</el-tag>
                </div>
                <div class="format-list" v-else-if="currentView === 'album'">
                  <el-tag class="format-item" type="warning">music.163.com/album?id=xxx</el-tag>
                  <el-tag class="format-item" type="warning">music.163.com/album/xxx</el-tag>
                  <el-tag class="format-item" type="warning">music.163.com/#/album?id=xxx</el-tag>
                  <el-tag class="format-item" type="warning">music.163.com/#/album/xxx</el-tag>
                  <el-tag class="format-item" type="warning">y.music.163.com/m/album/xxx</el-tag>
                </div>
              </div>
            </el-card>
          </div>

          <el-card class="author-card" shadow="hover">
            <template #header>
              <div class="card-header">
                <el-icon><User /></el-icon>
                <span>作者与联系</span>
              </div>
            </template>
            <div class="author-info">
              <p><strong>作者：</strong> 苏晓晴</p>
              <p><strong>个人博客：</strong> <el-link type="primary" href="https://www.toubiec.cn" target="_blank">点击跳转</el-link></p>
              <p><strong>联系方式：</strong> QQ3074193836</p>
              <p><strong>版权声明：</strong> 本工具仅用于学习交流，不提供任何音频文件存储服务。</p>
              <p><strong>反馈地址：</strong> <el-link type="primary" href="https://github.com/Suxiaoqinx/Netease_url/issues" target="_blank">点击跳转</el-link></p>
              <p>如果您在使用中遇到任何问题，或有任何建议，欢迎联系！</p>
            </div>
          </el-card>
        </div>
      </el-aside>
    </el-container>

    <!-- 底部 -->
    <el-footer class="app-footer">
      <div class="footer-content">
        <div class="footer-info">
          <div class="footer-links">
            <el-space :size="20" wrap>
              <el-link href="https://github.com/Suxiaoqinx/Netease_url" target="_blank" :underline="false">
                <el-icon><svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24"><path fill="currentColor" d="M12 2A10 10 0 0 0 2 12c0 4.42 2.87 8.17 6.84 9.5c.5.08.66-.23.66-.5v-1.69c-2.77.6-3.36-1.34-3.36-1.34c-.46-1.16-1.11-1.47-1.11-1.47c-.91-.62.07-.6.07-.6c1 .07 1.53 1.03 1.53 1.03c.87 1.52 2.34 1.07 2.91.83c.09-.65.35-1.09.63-1.34c-2.22-.25-4.55-1.11-4.55-4.92c0-1.11.38-2 1.03-2.71c-.1-.25-.45-1.29.1-2.64c0 0 .84-.27 2.75 1.02c.79-.22 1.65-.33 2.5-.33s1.71.11 2.5.33c1.91-1.29 2.75-1.02 2.75-1.02c.55 1.35.2 2.39.1 2.64c.65.71 1.03 1.6 1.03 2.71c0 3.82-2.34 4.66-4.57 4.91c.36.31.69.92.69 1.85V21c0 .27.16.59.67.5C19.14 20.16 22 16.42 22 12A10 10 0 0 0 12 2"/></svg></el-icon>
                GitHub
              </el-link>
              <el-link href="https://music.163.com" target="_blank" :underline="false">
                <el-icon><svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24"><path fill="none" stroke="currentColor" stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 4c-2.93 1.346-5 5.046-5 8.492C4 17 8 20 12 20s8-3 8-7c0-3.513-3.5-5.513-6-5.513S9 9 9 12c0 2 1.5 3 3 3s3-1 3-3c0-3.513-2-4.508-2-6.515c0-3.504 3.5-2.603 4-1.502"/></svg></el-icon>
                网易云音乐
              </el-link>
              <el-link href="https://element-plus.org" target="_blank" :underline="false">
                <el-icon><ElementPlus /></el-icon>
                Element Plus
              </el-link>
            </el-space>
          </div>
          
          <el-divider />
          
          <div class="footer-text">
            <div class="tech-stack">
              <el-tag size="small" type="success">Vue 3</el-tag>
              <el-tag size="small" type="primary">Element Plus</el-tag>
              <el-tag size="small" type="warning">Vite</el-tag>
              <el-tag size="small" type="info">JavaScript</el-tag>
            </div>
            
            <div class="copyright">
              © 2024 - 2025 网易云音乐无损解析 - 仅供学习交流使用
            </div>
            
              <div class="build-info">
              <el-text size="small" type="info">
        构建时间: 2025/11/29 | 版本: v1.1.9
              </el-text>
            </div>
          </div>
        </div>
      </div>
    </el-footer>
    
  </el-container>

  <!-- 设置弹窗 -->
  <teleport to="body">
    <el-dialog
      v-model="showSettingsDialog"
      title="网站设置"
      :width="isMobile ? '100%' : '600px'"
      :fullscreen="isMobile"
      :close-on-click-modal="true"
      :show-close="true"
      class="settings-dialog"
    >
    <div class="settings-content">
      <!-- 主题设置 -->
      <div class="setting-section">
        <!-- 主题外观设置已移除，改为右下角悬浮按钮切换 -->
        
        <!-- UI布局切换：兼容 Element Plus 表单写法 -->
        <el-form :model="settings" label-width="120px" label-position="left" class="settings-form">
          <el-form-item label="PC端布局模式">
            <el-select
              v-model="settings.layoutMode"
              @change="() => { saveSettings(); ElMessage.success('布局模式已保存'); }"
              style="width: 160px;"
            >
              <el-option label="双栏" value="dual-column" />
              <el-option label="单栏" value="single-column" />
            </el-select>
            <div class="form-item-hint">
              <el-text type="info" size="small">仅在PC端生效；用于切换双栏或单栏布局</el-text>
            </div>
          </el-form-item>
        </el-form>
      </div>

      <!-- 下载设置 -->
      <div class="setting-section">
        <div class="section-title">
          <el-icon><Download /></el-icon>
          <span>下载配置</span>
        </div>
        
        <!-- 使用 Element Plus 表单布局重构下载配置 -->
        <el-form :model="settings" label-width="120px" label-position="left" class="settings-form">
          <!-- 文件命名格式（下拉选择） -->
          <el-form-item label="文件命名格式">
            <el-select
              v-model="settings.filenameFormat"
              placeholder="请选择命名格式"
              style="width: 160px"
              @change="() => { saveSettings(); ElMessage.success('文件命名格式已保存'); }"
            >
              <el-option label="歌曲名 - 歌手名" value="song-artist" />
              <el-option label="歌手名 - 歌曲名" value="artist-song" />
              <el-option label="仅歌曲名" value="song-only" />
            </el-select>
            <div class="form-item-hint">
              <el-text type="info" size="small">
                示例：{{ settings.filenameFormat === 'artist-song' ? '周杰伦 - 夜曲' : (settings.filenameFormat === 'song-only' ? '夜曲' : '夜曲 - 周杰伦') }}
              </el-text>
            </div>
          </el-form-item>

          <!-- 自动写入元数据 -->
          <el-form-item label="自动写入元数据">
            <el-switch
              v-model="settings.writeMetadata"
              @change="() => { saveSettings(); ElMessage.success('元数据设置已保存'); }"
            />
            <div class="form-item-hint">
              <el-text type="info" size="small">写入歌曲名、歌手、专辑、封面等信息（不支持杜比全景声）</el-text>
            </div>
          </el-form-item>

          <!-- 启用 ZIP 打包 -->
          <el-form-item label="启用ZIP打包">
            <el-switch
              v-model="settings.zipDownload"
              @change="() => { saveSettings(); ElMessage.success('ZIP打包设置已保存'); }"
            />
            <div class="form-item-hint">
              <el-text type="info" size="small">打包音频、歌词、封面、信息文件为一个压缩包</el-text>
            </div>
          </el-form-item>

          <!-- 歌词格式选择：使用开关在 LRC/SRT 间切换 -->
          <el-form-item label="歌词格式">
            <el-switch
              v-model="settings.srtLyricsDownload"
              @change="() => { saveSettings(); ElMessage.success('歌词格式设置已保存'); }"
              active-text="SRT"
              inactive-text="LRC"
            />
            <div class="form-item-hint">
              <el-text type="info" size="small">开启为 SRT 字幕；关闭为 LRC 歌词</el-text>
            </div>
          </el-form-item>
        </el-form>
        
      </div>
      
      <!-- 解析设置 -->
      <div class="setting-section">
        <div class="section-title">
          <el-icon><List /></el-icon>
          <span>解析配置</span>
        </div>
        <!-- 解析与缓存配置统一为 Element Plus 表单 -->
        <el-form :model="settings" label-width="120px" label-position="left" class="settings-form">
          

          <!-- 播放链接缓存开关 -->
          <el-form-item label="启用链接缓存">
            <el-switch
              v-model="settings.enableUrlCache"
              @change="() => { saveSettings(); ElMessage.success('链接缓存设置已保存'); }"
            />
            <div class="form-item-hint">
              <el-text type="info" size="small">减少重复解析，提升相同歌曲再次播放速度</el-text>
            </div>
          </el-form-item>

          <!-- 环境信息 -->
          <el-form-item label="环境与版本">
            <div style="display:flex; align-items:center; gap:8px; flex-wrap:wrap;">
              <el-tag :type="isLocalEnv ? 'success' : 'info'" size="small">
                {{ isLocalEnv ? 'Dev环境' : 'Prod环境' }}
              </el-tag>
              
            </div>
          </el-form-item>

          

          
        </el-form>

      </div>
    </div>
    <template #footer>
      <div class="dialog-footer">
        <el-button type="primary" @click="saveSettings(); showSettingsDialog = false" :icon="Check">完成</el-button>
      </div>
    </template>
    </el-dialog>
  </teleport>

  <!-- 系统公告弹窗 -->
  <el-dialog
    v-model="showWelcomeDialog"
    title="系统公告"
    width="500px"
    :close-on-click-modal="false"
    :close-on-press-escape="false"
    :show-close="false"
    center
    class="welcome-dialog"
  >
    <div class="welcome-content">
      <div class="welcome-icon">
        <el-icon :size="48" color="#409EFF">
          <InfoFilled />
        </el-icon>
      </div>
      <h3 class="welcome-title">欢迎使用网易云音乐解析系统！</h3>
      <div class="welcome-message">
        <p>如果无法下载或下载后找不到文件请更换浏览器下载!</p>
      </div>
    </div>
    <template #footer>
       <div class="dialog-footer">
         <el-button type="primary" @click="handleWelcomeConfirm" size="large">
           我知道了
         </el-button>
       </div>
     </template>
  </el-dialog>

  <!-- 悬浮按钮 -->
  <!-- 主题切换按钮 -->
  <el-button
    class="floating-theme-btn floating-fab"
    type="default"
    circle
    size="large"
    @click="toggleThemeWithLoading()"
    :title="isDark ? '切换为浅色' : '切换为深色'"
  >
    <template #icon>
      <el-icon :size="22">
        <component :is="isDark ? Sunny : Moon" />
      </el-icon>
    </template>
  </el-button>

  

  

  

  

  

  

</el-config-provider>
</template>

<style scoped>
/* CSS变量定义 - 从style.css适配 */
:root {
  --el-color-primary: #409eff;
  --el-color-primary-light-3: #79bbff;
  --el-color-primary-light-5: #a0cfff;
  --el-color-primary-light-7: #c6e2ff;
  --el-color-primary-light-8: #d9ecff;
  --el-color-primary-light-9: #ecf5ff;
  --el-color-primary-dark-2: #337ecc;
  --el-color-success: #67c23a;
  --el-color-warning: #e6a23c;
  --el-color-danger: #f56c6c;
  --el-color-info: #909399;
  --el-bg-color: #fff;
  --el-bg-color-page: #f2f3f5;
  --el-text-color-primary: #303133;
  --el-text-color-regular: #606266;
  --el-text-color-secondary: #909399;
  --el-text-color-placeholder: #a8abb2;
  --el-border-color: #dcdfe6;
  --el-border-color-light: #e4e7ed;
  --el-border-color-lighter: #ebeef5;
  --el-fill-color-blank: #fff;
  --el-fill-color-light: #f5f7fa;
  --el-box-shadow: 0px 12px 32px 4px rgba(0,0,0,.04), 0px 8px 20px rgba(0,0,0,.08);
  --el-box-shadow-light: 0px 0px 12px rgba(0,0,0,.12);
  --el-transition-duration: .3s;
  --el-border-radius-base: 4px;
  --el-font-size-base: 14px;
  --el-font-size-large: 18px;
  --el-font-weight-primary: 500;
}

/* 深色模式变量 */
.dark {
  --el-bg-color: #141414;
  --el-bg-color-page: #0a0a0a;
  --el-text-color-primary: #e5eaf3;
  --el-text-color-regular: #cfd3dc;
  --el-text-color-secondary: #a3a6ad;
  --el-border-color: #4c4d4f;
  --el-border-color-light: #414243;
  --el-border-color-lighter: #363637;
  --el-fill-color-blank: #141414;
  --el-fill-color-light: #1d1e1f;
  --el-fill-color: #262727;
  --el-fill-color-darker: #1a1a1a;
  --el-color-primary-light-9: #1a1a2e;
  --el-color-primary: #409eff;
}

.title-card {
    margin-top: var(--space-4);
    margin-bottom: var(--space-4);
    text-align: center;
    background: var(--el-bg-color);
    border: 1px solid var(--el-border-color-lighter);
    border-radius: 16px
}

/* 应用容器 */
.app-container {
  min-height: 100vh;
  background: var(--el-bg-color-page);
  flex-direction: column;
  color: var(--el-text-color-primary);
  transition: all var(--el-transition-duration) ease;
}

/* 顶部导航栏样式 */
.app-header {
  height: 64px;
  background: var(--el-bg-color);
  border-bottom: 1px solid var(--el-border-color-light);
  box-shadow: var(--el-box-shadow-light);
  position: sticky;
  top: 0;
  z-index: 1000;
  transition: all var(--el-transition-duration) ease;
}

.header-content {
  width: 100%;
  max-width: 1400px;
  margin: 0 auto;
  padding: 0 var(--app-padding-x);
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.header-left {
  display: flex;
  align-items: center;
}

.logo {
  display: flex;
  align-items: center;
  gap: var(--space-2);
  cursor: pointer;
  transition: all var(--el-transition-duration) ease;
}

.logo:hover {
  transform: translateY(-1px);
}

.logo-icon {
  color: var(--el-color-primary);
  transition: all var(--el-transition-duration) ease;
}

.logo:hover .logo-icon {
  color: var(--el-color-primary-light-3);
  transform: rotate(5deg);
}

.logo-text {
  font-size: 20px;
  font-weight: 700;
  color: var(--el-text-color-primary);
  background: linear-gradient(135deg, var(--el-color-primary), var(--el-color-primary-light-3));
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
  transition: all var(--el-transition-duration) ease;
}

.header-right {
  display: flex;
  align-items: center;
  gap: var(--space-3);
}

.header-view-title {
  margin-left: var(--space-2);
  font-size: 16px;
  font-weight: 600;
  color: var(--el-text-color-primary);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.settings-btn {
  width: 48px;
  height: 48px;
  border-radius: 50%;
  background: var(--el-fill-color-light);
  border: 1px solid var(--el-border-color-lighter);
  color: var(--el-text-color-regular);
  transition: all var(--el-transition-duration) ease;
}

.settings-btn:hover {
  background: var(--el-color-primary-light-9);
  border-color: var(--el-color-primary-light-5);
  color: var(--el-color-primary);
  transform: translateY(-2px) rotate(90deg);
  box-shadow: var(--el-box-shadow);
}

.header-right .el-button {
  transition: all 0.3s ease;
}

.header-right .el-button:hover {
  transform: translateY(-1px);
  box-shadow: var(--el-box-shadow);
}

/* 主要内容 */
.main-container {
  max-width: 1200px;
  margin: 0 auto;
  width: 100%;
  flex: 1; /* 撑满剩余高度，确保 footer 贴底 */
}

.playlist-section {
  width: 100%;
}

.app-main {
  padding: var(--app-padding-x);
  display: flex;
  flex-direction: column;
  gap: var(--app-gap);
}

.app-main > .el-card, .content-grid .el-card {
  border-radius: 12px;
  width: 100%;
}

/* 页面标题 */
.page-header-card {
  text-align: center;
  background: var(--el-bg-color);
}

.page-header {
  text-align: center;
}

.page-title {
  font-size: 32px;
  font-weight: 700;
  color: var(--el-text-color-primary);
  margin: 0 0 10px 0;
}

.page-description {
  font-size: 16px;
  color: var(--el-text-color-regular);
  margin: 0;
}

/* 卡片式切换器样式 */
/* .view-switcher-container is now spaced by main-column gap */

.switcher-cards {
  display: flex;
  justify-content: center;
  gap: var(--app-gap);
  flex-wrap: wrap;
}

.switcher-card {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  width: 140px;
  height: 120px;
  padding: var(--space-4);
  background: var(--el-fill-color-blank);
  border: 2px solid var(--el-border-color-lighter);
  border-radius: 12px;
  cursor: pointer;
  transition: all 0.3s ease;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.switcher-card:hover {
  border-color: var(--el-color-primary-light-5);
  box-shadow: 0 4px 16px rgba(64, 158, 255, 0.2);
  transform: translateY(-2px);
}

.switcher-card.active {
  border-color: var(--el-color-primary);
  background: var(--el-color-primary-light-9);
  box-shadow: 0 4px 16px rgba(64, 158, 255, 0.3);
}

.card-icon {
  margin-bottom: var(--space-2);
  color: var(--el-text-color-secondary);
  transition: color 0.3s ease;
}

.switcher-card:hover .card-icon,
.switcher-card.active .card-icon {
  color: var(--el-color-primary);
}

.card-title {
  font-size: 14px;
  font-weight: 500;
  color: var(--el-text-color-primary);
  text-align: center;
  line-height: 1.4;
}

/* 按钮式切换样式 */
.view-buttons {
  display: flex;
  gap: var(--space-2);
  flex-wrap: wrap;
  justify-content: center;
}

.view-btn {
  min-width: 112px;
  padding: var(--space-1) var(--space-3);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.08);
}

.view-btn.el-button.is-round {
  border-radius: 999px;
}

.view-btn.el-button--primary {
  box-shadow: 0 6px 16px rgba(64, 158, 255, 0.3);
}

.dark .view-btn {
  background-color: var(--el-fill-color-blank);
  border-color: var(--el-border-color);
}

.dark .view-btn.el-button--primary {
  background-color: var(--el-color-primary);
  color: #fff;
}

/* 卡片样式 */
.input-card {
  /* margin-bottom is handled by parent gap */
  border-radius: 12px;
  transition: all var(--el-transition-duration) ease;
}

.input-card:hover {
  box-shadow: var(--el-box-shadow);
  transform: translateY(-2px);
}

.card-header {
  display: flex;
  align-items: center;
  gap: 8px;
  font-weight: 600;
}

/* 帮助区域 */
/* .help-section is now spaced by side-column gap */

.examples-section,
.formats-section {
  margin-bottom: var(--app-gap);
}

.examples-section:last-child,
.formats-section:last-child {
  margin-bottom: 0;
}

.section-title {
  font-size: 16px;
  font-weight: 600;
  color: var(--el-text-color-primary);
  margin: 0 0 var(--space-3) 0;
}

.example-list {
  display: flex;
  flex-direction: column;
  gap: var(--space-2);
}

.example-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: var(--space-2);
  border: 1px solid var(--el-border-color-lighter);
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.3s;
}

.example-item:hover {
  border-color: var(--el-color-primary);
  background-color: var(--el-color-primary-light-9);
}

.example-info {
  flex: 1;
  /* 允许内部文本正确收缩以实现省略号 */
  min-width: 0;
}

.example-name {
  font-weight: 600;
  color: var(--el-text-color-primary);
  margin-bottom: 4px;
}

.example-artist {
  font-size: 14px;
  color: var(--el-text-color-regular);
  /* 防止超长歌手名撑开布局 */
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.format-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.format-item {
  font-family: 'Courier New', monospace;
  font-size: 13px;
  align-self: flex-start;
}

/* 播放器区域 */
/* .player-section is now spaced by main-column gap */

/* 视图切换标签样式 */
.view-tabs {
  margin-bottom: var(--space-4);
}

.view-tabs .el-tabs__header {
  margin: 0 0 var(--space-4) 0;
}

.view-tabs .el-tabs__nav-wrap::after {
  height: 1px;
  background-color: var(--el-border-color-light);
}

.view-tabs .el-tabs__item {
  padding: 0 var(--space-4);
  height: 48px;
  line-height: 48px;
  font-size: 16px;
  font-weight: 500;
  color: var(--el-text-color-regular);
  transition: all 0.3s ease;
}

.view-tabs .el-tabs__item:hover {
  color: var(--el-color-primary);
}

/* 接口切换延迟显示 */
.btn-latency {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  margin-left: 6px;
  padding: 0 6px;
  border-radius: 8px;
  font-size: 12px;
  line-height: 1;
  transition: color var(--el-transition-duration) ease;
}
.btn-latency .dot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  display: inline-block;
}
.latency-chip {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  margin-left: 6px;
  font-size: 12px;
  line-height: 1;
}
.latency-low {
  color: var(--el-color-success);
}
.latency-low .dot {
  background-color: var(--el-color-success);
}
.latency-medium {
  color: var(--el-color-warning);
}
.latency-medium .dot {
  background-color: var(--el-color-warning);
}
.latency-high {
  color: var(--el-color-danger);
}
.latency-high .dot {
  background-color: var(--el-color-danger);
}
.latency-none {
  color: var(--el-text-color-secondary);
}
.latency-none .dot {
  background-color: var(--el-text-color-secondary);
}

.view-tabs .el-tabs__item.is-active {
  color: var(--el-color-primary);
  font-weight: 600;
}

.view-tabs .el-tabs__active-bar {
  height: 3px;
  background-color: var(--el-color-primary);
  border-radius: 2px;
}

.tab-label {
  display: flex;
  align-items: center;
  gap: 8px;
}

.tab-label .el-icon {
  font-size: 18px;
}

/* 歌单解析区域样式 */

/* .disclaimer-card is now spaced by app-main gap */

.disclaimer-card .el-alert {
  border-radius: 8px;
}

/* .author-card is now spaced by side-column gap */

.author-info p {
  margin: 0 0 var(--space-2) 0;
  font-size: 14px;
  color: var(--el-text-color-regular);
  line-height: 1.7;
}

.author-info p:last-child {
  margin-bottom: 0;
}

/* 设置区域样式 */
.settings-card {
  margin-top: var(--space-4);
  border-radius: 12px;
  overflow: hidden;
}

.card-header {
  display: flex;
  align-items: center;
  gap: var(--space-2);
}

.header-icon {
  font-size: 20px;
  color: var(--el-color-primary);
}

.header-title {
  font-size: 18px;
  font-weight: 600;
  color: var(--el-text-color-primary);
}

.settings-content {
  padding: var(--space-4);
  display: flex;
  flex-direction: column;
}

/* 设置弹窗中的切换按钮延迟显示样式 */
.btn-latency {
  margin-left: 6px;
  font-size: 12px;
}
.latency-ok {
  color: var(--el-color-success);
}
.latency-bad {
  color: var(--el-color-danger);
}
.latency-none {
  color: var(--el-text-color-secondary);
}

/* Element Plus 表单优化样式 */
.settings-form .el-form-item {
  margin-bottom: var(--space-3);
}
.settings-form :deep(.el-form-item__content) {
  display: grid;
  grid-auto-flow: row;
  align-items: flex-start;
}

.form-item-hint {
  margin-top: 6px;
}

/* 全局穿透：统一将所有 Element Plus 表单内容容器改为网格布局 */
:deep(.el-form-item__content) {
  display: grid;
}

/* 设置区块样式 */
.setting-section {
  display: flex;
  flex-direction: column;
  gap: var(--space-4);
}

.section-title {
  display: flex;
  align-items: center;
  gap: var(--space-1);
  font-size: 16px;
  font-weight: 600;
  color: var(--el-text-color-primary);
  margin-bottom: var(--space-3);
}

.section-title .el-icon {
  font-size: 18px;
  color: var(--el-color-primary);
}

/* 主题选项样式 */
.theme-options {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: var(--space-2);
}

.theme-option {
  display: flex;
  /*flex-direction: column;*/
  align-items: center;
  justify-content: center;
  padding: 0px 6px;
  border-radius: 12px;
  cursor: pointer;
  transition: all 0.3s ease;
  background: var(--el-bg-color);
  text-align: center;
  aspect-ratio: 1;
  min-height: 120px;
}

.theme-option:hover {
  border-color: var(--el-color-primary-light-5);
  background: var(--el-color-primary-light-9);
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}

.theme-option.active {
  border-color: var(--el-color-primary);
  background: var(--el-color-primary-light-9);
  box-shadow: 0 0 0 1px var(--el-color-primary-light-7);
}

.option-icon {
  font-size: 32px;
  color: var(--el-color-primary);
  margin-bottom: var(--space-2);
  flex-shrink: 0;
}

.option-content {
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
}

.option-label {
  font-size: 15px;
  font-weight: 600;
  color: var(--el-text-color-primary);
  margin-bottom: 4px;
}

.option-desc {
  font-size: 12px;
  color: var(--el-text-color-secondary);
  line-height: 1.3;
}

/* 设置组样式 */
.setting-group {
  display: flex;
  flex-direction: column;
  gap: var(--space-2);
}

.group-title {
  font-size: 14px;
  font-weight: 500;
  color: var(--el-text-color-primary);
  margin-bottom: var(--space-2);
}

/* 文件命名选项 */
.filename-options {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.filename-option {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: var(--space-3);
  border: 2px solid var(--el-border-color-light);
  border-radius: 12px;
  cursor: pointer;
  transition: all 0.3s ease;
  background: var(--el-bg-color);
}

.filename-option:hover {
  border-color: var(--el-color-primary-light-5);
  background: var(--el-color-primary-light-9);
  transform: translateY(-1px);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.filename-option.active {
  border-color: var(--el-color-primary);
  background: var(--el-color-primary-light-9);
  box-shadow: 0 0 0 1px var(--el-color-primary-light-7);
}

.form-items-row {
    display: flex;
    gap: var(--space-4);
    align-items: center;
}

.option-content {
  flex: 1;
}

.option-header {
  display: flex;
  align-items: center;
  gap: var(--space-1);
  margin-bottom: 4px;
}

.option-title {
  font-size: 14px;
  font-weight: 500;
  color: var(--el-text-color-primary);
}

.option-example {
  font-size: 12px;
  color: var(--el-text-color-secondary);
  font-family: 'Monaco', 'Menlo', 'Ubuntu Mono', monospace;
}

.option-indicator {
  display: flex;
  align-items: center;
}

.radio-dot {
  width: 16px;
  height: 16px;
  border: 2px solid var(--el-border-color);
  border-radius: 50%;
  position: relative;
  transition: all 0.3s ease;
}

.radio-dot::after {
  content: '';
  position: absolute;
  top: 50%;
  left: 50%;
  width: 6px;
  height: 6px;
  background: var(--el-color-primary);
  border-radius: 50%;
  transform: translate(-50%, -50%) scale(0);
  transition: transform 0.3s ease;
}

.filename-option.active .radio-dot {
  border-color: var(--el-color-primary);
}

.filename-option.active .radio-dot::after {
  transform: translate(-50%, -50%) scale(1);
}

/* 开关卡片样式 */
.switch-card {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: var(--space-3);
  border: 2px solid var(--el-border-color-light);
  border-radius: 12px;
  background: var(--el-bg-color);
  transition: all 0.3s ease;
  /* margin-bottom is handled by parent gap */
}

.switch-card:hover {
  border-color: var(--el-color-primary-light-5);
  background: var(--el-color-primary-light-9);
  transform: translateY(-1px);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.switch-content {
  flex: 1;
  margin-right: var(--space-3);
}

.switch-header {
  display: flex;
  align-items: flex-start;
  gap: var(--space-2);
}

.switch-icon {
  font-size: 18px;
  color: var(--el-color-primary);
  margin-top: 2px;
  flex-shrink: 0;
}

.switch-info {
  flex: 1;
}

.switch-title {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 15px;
  font-weight: 500;
  color: var(--el-text-color-primary);
  margin-bottom: 4px;
}

.switch-desc {
  font-size: 13px !important;
  color: var(--el-text-color-secondary) !important;
  line-height: 1.4 !important;
  display: block !important;
  visibility: visible !important;
  opacity: 1 !important;
}

.switch-control {
  flex-shrink: 0;
}

/* 操作按钮区域 */
.action-section {
  padding-top: 24px;
  border-top: 1px solid var(--el-border-color-lighter);
  margin-top: 24px;
}

.button-group {
  display: flex;
  gap: var(--space-2);
  justify-content: flex-end;
}

.reset-btn,
.save-btn {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: var(--space-2) calc(var(--space-3) + var(--space-1));
  border-radius: 8px;
  font-weight: 500;
  min-height: 44px;
  transition: all 0.3s ease;
}

.reset-btn:hover,
.save-btn:hover {
  transform: translateY(-1px);
}

.btn-text {
  white-space: nowrap;
}

/* 桌面端设置抽屉头部样式 */
.desktop-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  width: 100%;
}

/* 手机端设置抽屉样式 */
.settings-drawer .el-drawer__header {
    margin-bottom: 0px;
    padding-bottom: var(--space-4);
}
.settings-drawer .el-drawer__body {
    padding-top: 0px;
}

/* 手机端顶部栏 */
.mobile-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: var(--space-3) var(--space-4);
  background: var(--el-bg-color);
  border-bottom: 1px solid var(--el-border-color-light);
  position: sticky;
  top: 0;
  z-index: 10;
}

.mobile-title {
  font-size: 18px;
  font-weight: 600;
  color: var(--el-text-color-primary);
}

.mobile-done-btn {
  min-width: 80px;
  height: 36px;
  border-radius: 18px;
  font-weight: 500;
}

/* 手机端弹窗全屏样式 */
@media (max-width: 768px) {
  .settings-drawer .el-drawer__body {
    padding: 0 !important;
  }

  .settings-drawer .el-drawer__header {
    padding: 0 !important;
    margin: 0 !important;
  }
    
  .settings-content {
    padding: var(--space-4) !important;
  }
}

/* 响应式设计 */
@media (max-width: 768px) {
  .settings-content {
    padding: var(--space-3);
  }
  
  .theme-options {
    display: flex;
    flex-direction: column;
    gap: var(--space-1);
  }
  
  .theme-option {
    display: flex;
    flex-direction: row;
    align-items: center;
    text-align: left;
    padding: var(--space-3);
    min-height: 60px;
    border-radius: 12px;
    aspect-ratio: unset;
  }
  
  .theme-option:active {
    transform: scale(0.98);
    transition: transform 0.1s ease;
  }
  
  .option-icon {
    font-size: 20px;
    margin-bottom: 0;
    margin-right: var(--space-2);
    flex-shrink: 0;
  }
  
  .option-content {
    display: block;
    text-align: left;
  }
  
  .option-label {
    font-size: 16px;
    margin-bottom: 2px;
  }
  
  .option-desc {
    font-size: 14px;
  }
  
  .switch-card {
    /* flex-direction: column; */
    align-items: center;
    gap: var(--space-2);
    padding: var(--space-3);
    min-height: 60px;
    border-radius: 12px;
  }
  
  .switch-card:active {
    transform: scale(0.98);
    transition: transform 0.1s ease;
  }
  
  .switch-content {
    margin-right: 0;
    /* width: 100%; */
  }
  
  .switch-header {
    gap: var(--space-1);
  }
  
  .filename-option {
    padding: var(--space-3);
    min-height: 60px;
    border-radius: 12px;
  }
  
  .filename-option:active {
    transform: scale(0.98);
    transition: transform 0.1s ease;
  }
  
  .button-group {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: var(--space-2);
  }
  
  .reset-btn,
  .save-btn {
    width: 100%;
    justify-content: center;
    min-height: 44px;
    font-size: 14px;
    padding: var(--space-2) var(--space-3);
  }
}

@media (max-width: 480px) {
  /* 卡片切换器移动端适配 */
  .switcher-cards {
    gap: var(--space-2);
  }
  
  .switcher-card {
    width: 100px;
    height: 100px;
    padding: var(--space-3);
  }
  
  .card-icon {
    margin-bottom: 8px;
  }
  
  .card-icon .el-icon {
    font-size: 20px !important;
  }
  
  .card-title {
    font-size: 12px;
  }

.form-items-row {
    display: flex;
    flex-direction: column;
    gap: 0;
}

  .settings-content {
    padding: var(--space-2);
  }
  
  .section-title {
    font-size: 15px;
  }
  
  .theme-option {
    padding: var(--space-1) var(--space-2);
  }
  
  .option-label {
    font-size: 13px;
  }
  
  .option-desc {
    font-size: 11px;
  }
  
  .item-label {
    font-size: 14px;
  }
  
  .item-desc {
    font-size: 12px;
  }
  
  .filename-option {
    padding: var(--space-1) var(--space-2);
  }
  
  .option-title {
    font-size: 13px;
  }
  
  .option-example {
    font-size: 11px;
  }
  
  .switch-card {
    padding: var(--space-1) var(--space-2);
  }
  
  .switch-title {
    font-size: 14px;
  }
  
  .switch-desc {
    font-size: 12px;
  }
  
  .switch-icon {
    font-size: 16px;
  }
  
  .button-group {
    gap: 8px;
  }
  
  .reset-btn,
  .save-btn {
    padding: 12px 14px;
    font-size: 13px;
    min-height: 40px;
    border-radius: 8px;
  }

  .btn-text {
    font-size: 13px;
    font-weight: 500;
  }
}

/* 底部 */
.app-footer {
  width: 100%;
  background-color: var(--el-bg-color);
  border-top: 1px solid var(--el-border-color-light);
  padding: var(--space-4) var(--space-4) var(--space-4);
  height: 100%;
}

.footer-content {
  width: 100%;
  padding: 0 var(--app-padding-x);
}

.footer-info {
  text-align: center;
}

.footer-links {
  margin-bottom: var(--app-gap);
}

.footer-links .el-link {
  color: var(--el-text-color-regular);
  font-weight: 500;
  transition: all 0.3s;
}

.footer-links .el-link:hover {
  color: var(--el-color-primary);
  transform: translateY(-1px);
}

.footer-links .el-icon {
  margin-right: 4px;
}

.footer-text {
  display: flex;
  flex-direction: column;
  gap: var(--space-2);
  align-items: center;
}

.tech-stack {
  display: flex;
  gap: var(--space-1);
  justify-content: center;
  flex-wrap: wrap;
}

.copyright {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
  font-size: 14px;
  color: var(--el-text-color-regular);
  font-weight: 500;
}

.build-info {
  opacity: 0.8;
}

/* 响应式设计 */
@media (max-width: 1024px) {
  .header-content {
    padding: 0 var(--app-padding-x);
  }

  .logo-text {
    font-size: 18px;
  }
}

@media (max-width: 768px) {
  .app-header {
    height: 56px;
  }

  .header-content {
    padding: 0 var(--app-padding-x);
  }
  
  .logo {
    gap: 10px;
  }
  
  .logo-text {
    font-size: 16px;
  }
  .header-view-title {
    font-size: 14px;
    max-width: 40vw;
  }
  
  .header-right .el-button {
    width: 40px;
    height: 40px;
  }
  
  .app-main {
    padding: var(--space-3);
  }
  /* 覆盖移动端作者卡片顶部外边距，使用外层 gap 控制 */
  .author-card {
    margin-top: 0;
  }
  
  .page-title {
    font-size: 24px;
  }
  
  .page-description {
    font-size: 14px;
  }
  
  .help-section .el-col {
    margin-bottom: var(--app-gap);
  }
  
  /* 输入表单移动端优化 */
  .input-card .el-form-item {
    display: flex!important;
    width: 100%!important;
    margin-left: 0!important;
    margin-right: 0!important
  }
  
  .input-card .el-input {
    font-size: 16px;
  }

  .input-card .el-input__inner {
    height: 44px;
    font-size: 16px;
  }
  
  .input-card .el-select {
    width: 100% !important;
  }
  
  .input-card .el-select .el-input__inner {
    height: 44px;
    font-size: 16px;
  }
  
  .input-card .el-button {
    height: 44px;
    font-size: 16px;
    min-width: 120px;
  }
  
  .input-card .el-form-item__label {
    font-size: 15px;
    font-weight: 500;
  }
  
  /* 设置区域手机端适配 */
  .settings-container {
    padding: var(--space-3);
  }
  
  .header-icon-wrapper {
    width: 40px;
    height: 40px;
  }
  
  .header-icon {
    font-size: 20px;
  }
  
  .header-title {
    font-size: 18px;
  }
  
  .header-subtitle {
    font-size: 13px;
  }
  
  .section-header {
    gap: var(--space-2);
    margin-bottom: var(--space-3);
  }
  
  .section-icon {
    width: 36px;
    height: 36px;
    font-size: 18px;
  }
  
  .section-title {
    font-size: 16px;
  }
  
  .section-description {
    font-size: 13px;
  }
  
  .theme-options {
    grid-template-columns: 1fr;
    gap: var(--space-2);
  }
  
  .theme-option {
    padding: var(--space-3);
  }
  
  .option-icon {
    width: 40px;
    height: 40px;
    font-size: 20px;
    margin-bottom: 10px;
  }
  
  .option-label {
    font-size: 15px;
  }
  
  .option-desc {
    font-size: 12px;
  }
  
  .filename-option {
    padding: 14px var(--space-3);
  }
  
  .filename-option .option-label {
    font-size: 14px;
  }
  
  .option-example {
    font-size: 12px;
  }
  
  .switch-setting {
    padding: var(--space-3);
    flex-direction: column;
    align-items: flex-start;
    gap: var(--space-2);
  }
  
  .switch-info {
    margin-right: 0;
    width: 100%;
  }
  
  .switch-label {
    font-size: 15px;
  }
  
  .switch-desc {
    font-size: 12px;
  }
  
  .custom-switch {
    align-self: flex-end;
  }
  
  .action-section {
    flex-direction: column;
    gap: var(--space-2);
  }
  
  .reset-button,
  .save-button {
    width: 100%;
    min-width: auto;
    height: 48px;
  }
  
  .app-footer {
    padding: var(--space-4) var(--space-3) var(--space-3);
  }
  
  .footer-links {
    margin-bottom: var(--app-gap);
  }
  
  .footer-links .el-space {
    justify-content: center;
    flex-wrap: wrap;
  }
  
  .footer-links .el-link {
    font-size: 14px;
    padding: var(--space-1) var(--space-2);
    border-radius: 6px;
    background-color: var(--el-fill-color-light);
    transition: all 0.3s;
  }
  
  .footer-links .el-link:hover {
    background-color: var(--el-color-primary-light-9);
  }
  
  .tech-stack {
    gap: var(--space-1);
    justify-content: center;
  }
  
  .tech-stack .el-tag {
    margin: 2px;
  }
  
  .footer-text {
    gap: var(--space-2);
  }
  
  .copyright {
    font-size: 13px;
    line-height: 1.4;
  }
  
  .build-info {
    font-size: 12px;
  }
}

@media (max-width: 480px) {
  .logo-text {
    display: none;
  }
  
  .app-main {
    padding: var(--space-1);
  }
  
  .page-header {
    margin-bottom: var(--space-4);
  }
  
  .page-title {
    font-size: 20px;
  }
  
  .page-description {
    font-size: 13px;
  }
  
  /* 输入表单小屏幕优化 */
  
  .input-card .el-form-item {
    margin-bottom: var(--app-gap);
  }
  
  .input-card .el-form-item__label {
    font-size: 14px;
    margin-bottom: 8px;
  }
  
  .input-card .el-input__inner,
  .input-card .el-select .el-input__inner {
    height: 40px;
    font-size: 15px;
  }
  
  .input-card .el-button {
    height: 40px;
    font-size: 15px;
    min-width: 100px;
    margin-right: 8px;
    margin-bottom: 8px;
  }
  
  /* 按钮组移动端布局 */
  .input-card .el-form-item:last-child {
    display: flex;
    flex-direction: column;
    gap: 8px;
  }
  
  .input-card .el-form-item:last-child .el-button {
     width: 100%;
     margin-right: 0;
   }
   
   /* 卡片移动端优化 */
   .el-card {
     border-radius: 12px;
     box-shadow: 0 2px 12px 0 rgba(0, 0, 0, 0.08);
   }
   
   .el-card .el-card__header {
     padding: var(--space-3);
   }
   
   .el-card .el-card__body {
     padding: var(--space-3);
   }
   
   .card-header {
     font-size: 16px;
     font-weight: 600;
   }
   
   /* 帮助区域移动端优化 */
  .help-section .el-card {
    margin-bottom: 0; /* 由外层栅格 gap 控制间距 */
  }
   
   .examples-section {
     display: flex;
     flex-direction: column;
     gap: var(--space-2);
   }
   
   .example-item {
     padding: var(--space-2);
     background: var(--el-fill-color-lighter);
     border-radius: 8px;
     font-size: 14px;
   }
   
.input-card .el-form-item:last-child {
    flex-direction: unset;
    gap: 0;
}

   /* 设置区域超小屏幕适配 */
  .settings-container {
    padding: var(--space-1);
  }
  
  .card-header {
    gap: var(--space-2);
  }
  
  .header-icon-wrapper {
    width: 36px;
    height: 36px;
  }
  
  .header-icon {
    font-size: 18px;
  }
  
  .header-title {
    font-size: 16px;
  }
  
  .header-subtitle {
    font-size: 12px;
  }
  
  .section-header {
    gap: 10px;
    margin-bottom: 14px;
  }
  
  .section-icon {
    width: 32px;
    height: 32px;
    font-size: 16px;
  }
  
  .section-title {
    font-size: 15px;
  }
  
  .section-description {
    font-size: 12px;
  }
  
  .theme-option {
    padding: var(--space-1);
  }
  
  .option-icon {
    width: 36px;
    height: 36px;
    font-size: 18px;
    margin-bottom: var(--space-1);
  }
  
  .option-label {
    font-size: 14px;
  }
  
  .option-desc {
    font-size: 11px;
  }
  
  .group-label {
    font-size: 15px;
  }
  
  .label-icon {
    font-size: 16px;
  }
  
  .filename-option {
    padding: var(--space-1) 14px;
  }
  
  .filename-option .option-label {
    font-size: 13px;
  }
  
  .option-example {
    font-size: 11px;
  }
  
  .switch-setting {
    padding: 14px;
  }
  
  .switch-label {
    font-size: 14px;
  }
  
  .switch-desc {
    font-size: 11px;
  }
  
  .switch-icon {
    font-size: 14px;
  }
  
  .reset-button,
  .save-button {
    height: 44px;
    font-size: 14px;
  }
  
  .btn-icon {
    font-size: 14px;
  }
  
  .footer-links .el-space {
    flex-direction: column;
    align-items: center;
    gap: 8px;
  }
  
  .footer-links .el-link {
    width: 200px;
    text-align: center;
    justify-content: center;
    font-size: 13px;
    padding: var(--space-1) var(--space-3);
  }
  
  .tech-stack {
    flex-wrap: wrap;
    justify-content: center;
    gap: 4px;
    max-width: 280px;
    margin: 0 auto;
  }
  
  .tech-stack .el-tag {
    margin: 2px;
    font-size: 11px;
  }
  
  .copyright {
    flex-direction: column;
    gap: 6px;
    text-align: center;
    font-size: 12px;
    line-height: 1.5;
  }
  
  .build-info {
    font-size: 11px;
    text-align: center;
  }
  
  .footer-text {
    gap: 8px;
  }
}

/* 触摸优化 */
@media (hover: none) and (pointer: coarse) {
  .theme-option,
  .filename-option {
    min-height: 64px;
  }
  
  .switch-card {
    min-height: 64px;
  }
  
  .mobile-done-btn {
    min-height: 44px;
    min-width: 88px;
  }
  
  .option-icon {
    min-width: 44px;
    min-height: 44px;
  }
  
  /* 增强触摸反馈 */
  .theme-option:active,
  .filename-option:active,
  .switch-card:active {
    transform: scale(0.97);
    background-color: var(--el-color-primary-light-8);
  }
  
  .mobile-done-btn:active {
    transform: scale(0.95);
  }
}

/* 手机端专用交互优化 */
@media (max-width: 768px) {
  /* 增加所有可点击元素的触摸区域 */
  .theme-option,
  .filename-option,
  .switch-card {
    position: relative;
    -webkit-tap-highlight-color: transparent;
    user-select: none;
  }
  
  /* 优化开关控件 */
  .switch-card .el-switch {
    transform: scale(1.2);
  }
  
  /* 优化单选按钮 */
  .radio-dot {
    width: 20px;
    height: 20px;
    border-width: 3px;
  }
  
  .radio-dot::after {
    width: 8px;
    height: 8px;
  }
  
  /* 移动端滚动优化 */
  .settings-content {
    -webkit-overflow-scrolling: touch;
    scroll-behavior: smooth;
  }
  
  /* 移动端字体优化 */
  .option-label,
  .switch-title {
    font-size: 16px;
    line-height: 1.4;
  }
  
  .option-desc,
  .switch-desc {
    font-size: 14px;
    line-height: 1.3;
  }
}

/* 深色模式适配 */
.dark .settings-card {
  background-color: var(--el-bg-color);
  border-color: var(--el-border-color);
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.15);
}

.dark .header-icon-wrapper {
  background: linear-gradient(135deg, var(--el-color-primary-dark-2), var(--el-color-primary));
  box-shadow: 0 4px 12px rgba(64, 158, 255, 0.4);
}

.dark .section-icon {
  background-color: var(--el-color-primary-light-8);
  color: var(--el-color-primary-light-3);
}

.dark .theme-option,
.dark .filename-option {
  background-color: var(--el-fill-color-blank);
  border-color: var(--el-border-color);
}

.dark .theme-option:hover,
.dark .filename-option:hover {
  border-color: var(--el-color-primary-light-5);
  background-color: var(--el-color-primary-light-9);
}

.dark .theme-option.active,
.dark .filename-option.active {
  border-color: var(--el-color-primary);
  background-color: var(--el-color-primary-light-9);
}

.dark .option-icon {
  color: var(--el-color-primary-light-3);
  background-color: transparent;
}

.dark .themer.switch-setting:hover {
  border-color: var(--el-color-primary-light-5);
  background-color: var(--el-color-primary-light-9);
}
/* 深色模式下的顶部导航栏 */
.dark .app-header {
  background: var(--el-bg-color);
  border-bottom-color: var(--el-border-color);
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.3);
}

.dark .logo-text {
  background: linear-gradient(135deg, var(--el-color-primary-light-3), var(--el-color-primary-light-5));
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.dark .settings-btn {
  background: var(--el-fill-color-light);
  border-color: var(--el-border-color);
  color: var(--el-text-color-regular);
}

.dark .settings-btn:hover {
  background: var(--el-color-primary-light-9);
  border-color: var(--el-color-primary-light-5);
  color: var(--el-color-primary-light-3);
  box-shadow: 0 4px 12px rgba(64, 158, 255, 0.3);
}

.dark .save-button:hover {
  background: linear-gradient(135deg, var(--el-color-primary-dark-2), var(--el-color-primary));
  box-shadow: 0 6px 20px rgba(64, 158, 255, 0.5);
}

.dark .section-divider {
  background: linear-gradient(90deg, transparent, var(--el-border-color), transparent);
}

/* 动画效果 */
@keyframes slideInUp {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.setting-section {
  animation: slideInUp 0.6s ease-out;
}

.setting-section:nth-child(2) {
  animation-delay: 0.1s;
}

.setting-section:nth-child(3) {
  animation-delay: 0.2s;
}

.setting-section:nth-child(4) {
  animation-delay: 0.3s;
}

/* 深色模式切换动画 */
* {
  transition: background-color 0.3s cubic-bezier(0.4, 0, 0.2, 1),
              color 0.3s cubic-bezier(0.4, 0, 0.2, 1),
              border-color 0.3s cubic-bezier(0.4, 0, 0.2, 1),
              box-shadow 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}

/* 全局盒模型，避免元素计算宽度时溢出 */
*, *::before, *::after {
  box-sizing: border-box;
}

/* 主要容器的过渡动画 */
.app-container,
.app-header,
.app-main,
.input-card,
.result-card {
  transition: background-color 0.4s cubic-bezier(0.4, 0, 0.2, 1),
              border-color 0.4s cubic-bezier(0.4, 0, 0.2, 1),
              box-shadow 0.4s cubic-bezier(0.4, 0, 0.2, 1);
margin-top:0;
}

/* Element Plus 组件的过渡动画 */
.el-card,
.el-button,
.el-input,
.el-select,
.el-radio,
.el-switch,
.el-dialog,
.el-message,
.el-notification {
  transition: background-color 0.3s cubic-bezier(0.4, 0, 0.2, 1),
              color 0.3s cubic-bezier(0.4, 0, 0.2, 1),
              border-color 0.3s cubic-bezier(0.4, 0, 0.2, 1) !important;
}

/* 主题切换时的特殊动画效果 */
.theme-switching {
  position: relative;
  overflow: hidden;
}

.theme-switching::before {
  content: '';
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: radial-gradient(circle at center, 
    var(--el-color-primary-light-9) 0%, 
    transparent 70%);
  opacity: 0;
  pointer-events: none;
  z-index: 9999;
  animation: themeRipple 0.6s cubic-bezier(0.4, 0, 0.2, 1);
}

@keyframes themeRipple {
  0% {
    opacity: 0;
    transform: scale(0);
  }
  50% {
    opacity: 0.3;
    transform: scale(1);
  }
  100% {
    opacity: 0;
    transform: scale(1.2);
  }
}

/* 主题选项的动画增强 */
.theme-option {
  transition: all 0.4s cubic-bezier(0.4, 0, 0.2, 1);
  position: relative;
  overflow: hidden;
}
.theme-option:hover {
  transform: translateY(-1px);
  box-shadow: var(--el-box-shadow);
}

.theme-option.active {
  border-color: var(--el-color-primary);
  background-color: var(--el-color-primary-light-9);
}

.welcome-dialog .el-dialog {
  width: 400px;
  margin: 0 auto;
}

.welcome-content {
  padding: 24px 0;
  text-align: center;
}

.welcome-icon {
  margin-bottom: 10px;
}

.welcome-message p {
  margin: 0;
  padding: var(--space-2) var(--space-4);
  background: var(--el-color-warning-light-9);
  border: 1px solid var(--el-color-warning-light-7);
  border-radius: 8px;
  color: var(--el-color-warning-dark-2);
}

.dialog-footer {
  text-align: center;
}

/* 欢迎弹窗移动端适配 */
@media (max-width: 768px) {
  .welcome-dialog .el-dialog {
    width: 90% !important;
    margin: 0 auto !important;
    border-radius: 12px !important;
  }
  
  .welcome-content {
    padding: var(--space-3) 0;
  }
  
  .welcome-icon {
    margin-bottom: var(--space-3);
  }
  
  .welcome-title {
    font-size: 18px;
    margin: 0 0 var(--space-3) 0;
  }
  
  .welcome-message {
    font-size: 15px;
    margin-bottom: 8px;
  }
  
  .welcome-message p {
    padding: var(--space-1) var(--space-3);
    font-size: 14px;
    line-height: 1.5;
  }
  
  .dialog-footer .el-button {
    width: 120px;
    height: 44px;
    font-size: 16px;
  }
}

@media (max-width: 480px) {
  .welcome-dialog .el-dialog {
    width: 95% !important;
    margin: 0 auto !important;
  }
  
  .welcome-content {
    padding: var(--space-2) 0;
  }
  
  .welcome-icon {
    margin-bottom: var(--space-2);
  }
  
  .welcome-icon .el-icon {
    font-size: 40px !important;
  }
  
  .welcome-title {
    font-size: 16px;
    margin: 0 0 var(--space-2) 0;
  }
  
  .welcome-message {
    font-size: 14px;
  }
  
  .welcome-message p {
    padding: var(--space-1) var(--space-2);
    font-size: 13px;
  }
  
  .dialog-footer .el-button {
    width: 100px;
    height: 40px;
    font-size: 14px;
  }
}

@keyframes bounce {
  0%, 20%, 50%, 80%, 100% {
    transform: translateY(0);
  }
  40% {
    transform: translateY(-10px);
  }
  60% {
    transform: translateY(-5px);
  }
}

/* 悬浮按钮样式 - 使用Element Plus标准样式 */
/* Removed changelog floating button styles */

/* 赞助按钮样式 - 使用Element Plus标准样式 */
/* Removed sponsor floating button styles */

/* 悬浮按钮统一白底/黑图标，夜间黑底/白图标 */
.floating-fab {
  background: #ffffff !important;
  color: #000000 !important;
  border: none !important;
}
.floating-fab .el-icon {
  color: #000000 !important;
}
.floating-fab:hover {
  transform: translateY(-2px);
  box-shadow: var(--el-box-shadow) !important;
  background: #f5f5f5 !important;
}
.floating-fab:active {
  transform: translateY(0);
}
.dark .floating-fab {
  background: #1a1a1a !important;
  color: #ffffff !important;
}
.dark .floating-fab .el-icon {
  color: #ffffff !important;
}

/* 主题切换按钮样式 - 与悬浮按钮同款 */
.floating-theme-btn {
  position: fixed;
  bottom: 168px; /* 在赞助按钮之上 */
  right: 24px;
  z-index: 1000;
  width: 56px;
  height: 56px;
  font-size: 20px;
  box-shadow: var(--el-box-shadow-light);
  transition: var(--el-transition-all);
}

.floating-theme-btn:hover {
  transform: translateY(-2px);
  box-shadow: var(--el-box-shadow);
}

/* 移除 API 列表按钮样式 */

.floating-theme-btn:active {
  transform: translateY(0);
}

@media (max-width: 768px) {
  .floating-theme-btn {
    bottom: 136px;
    right: 20px;
    width: 48px;
    height: 48px;
    font-size: 18px;
  }
}

@media (max-width: 480px) {
  .floating-theme-btn {
    bottom: 128px;
    right: 16px;
    width: 44px;
    height: 44px;
    font-size: 16px;
  }
}

 
/* Suspense 加载占位符样式 */
.loading-placeholder {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: calc(var(--space-4) * 2) var(--space-4);
  min-height: 200px;
  background: var(--el-bg-color-page);
  border-radius: 8px;
  border: 1px dashed var(--el-border-color-lighter);
}

.loading-text {
  margin-top: var(--space-3);
  color: var(--el-text-color-secondary);
  font-size: 14px;
}

/* 全屏Loading样式 */
.fullscreen-loading {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: rgba(0, 0, 0, 0.7);
  backdrop-filter: blur(4px);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 9999;
  animation: fadeIn 0.3s ease-in-out;
}

.loading-content {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  background: var(--el-bg-color);
  padding: 40px 60px;
  border-radius: 16px;
  box-shadow: var(--el-box-shadow-dark);
  text-align: center;
  min-width: 300px;
}

.loading-icon {
  color: var(--el-color-primary);
  animation: spin 1.5s linear infinite;
}

.fullscreen-loading .loading-text {
  margin-top: var(--space-4);
  font-size: 18px;
  font-weight: 600;
  color: var(--el-text-color-primary);
}

.loading-subtitle {
  margin-top: 8px;
  font-size: 14px;
  color: var(--el-text-color-secondary);
}

.loading-progress {
  margin-top: 10px;
  font-size: 13px;
  color: var(--el-color-primary);
  font-weight: 500;
  background: var(--el-color-primary-light-9);
  padding: 6px var(--space-2);
  border-radius: 4px;
  border: 1px solid var(--el-color-primary-light-7);
}

.loading-timer {
  margin-top: var(--space-2);
  font-size: 13px;
  color: var(--el-text-color-regular);
  font-family: 'Courier New', monospace;
  letter-spacing: 0.5px;
  opacity: 0.8;
}

@keyframes fadeIn {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}

@keyframes spin {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}

/* 深色模式适配 */
.dark .fullscreen-loading {
  background: rgba(0, 0, 0, 0.8);
}

/* 骨架屏动画优化 */
.loading-placeholder .el-skeleton {
  width: 100%;
  max-width: 400px;
}

/* 循环解析设置样式 */
.loop-parse-settings {
  margin-top: var(--space-3);
  padding: var(--space-3);
  background: var(--el-fill-color-lighter);
  border-radius: 8px;
  border: 1px solid var(--el-border-color-lighter);
}

.setting-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: var(--space-2);
}

.setting-item:last-child {
  margin-bottom: 0;
}

.setting-label {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 14px;
  font-weight: 500;
  color: var(--el-text-color-primary);
}

.setting-control {
  display: flex;
  align-items: center;
  gap: 8px;
}

.setting-unit {
  font-size: 14px;
  color: var(--el-text-color-secondary);
}

.setting-description {
  display: flex;
  align-items: flex-start;
  gap: 6px;
  font-size: 12px;
  color: var(--el-text-color-secondary);
  line-height: 1.4;
  margin-top: 8px;
}

.setting-description .el-icon {
  font-size: 14px;
  color: var(--el-color-info);
  margin-top: 1px;
  flex-shrink: 0;
}

/* 深色模式适配 */
.dark .loop-parse-settings {
  background: var(--el-fill-color-dark);
  border-color: var(--el-border-color);
}

/* PC端双栏布局 */
@media (min-width: 1025px) {

.main-content {
    flex: 1;
    padding: calc(var(--space-4) * 5) var(--space-4) var(--space-4);
    max-width: 1200px;
    margin: 0 auto;
    width: 100%;
}

  .side-column{
    position: sticky;
    top: 80px;
    /* 防止在grid中被拉伸，确保粘性生效 */
    align-self: start;
    height: fit-content;
  }
  
  .main-column, .side-column {
    display: flex;
    flex-direction: column;
    gap: var(--app-gap);
    /* 允许列在网格中正确收缩，避免被长文本撑开 */
    min-width: 0;
  }

  /* Margins are removed from base styles, gap is used instead */
  .view-switcher-container,
  .input-card,
  .player-section,
  .help-section,
  .author-card {
    margin-bottom: 0;
  }
}

/* 仅在桌面宽度下覆盖 el-main 的 overflow，避免移动端错位 */
@media (min-width: 1025px) {
  .el-main {
    overflow: visible;
  }
}

/* 移动端适配 */
@media (max-width: 768px) {
  .setting-item {
    flex-direction: column;
    align-items: flex-start;
    gap: 8px;
  }
}
  
  .content-grid.single-column-layout .side-column {
    margin-block: var(--space-4);
  }

  .setting-control {
    width: 100%;
    justify-content: space-between;
  }

  /* 防止栅格子项因内容过长溢出容器 */
  .content-grid > * {
    min-width: 0;
  }
  
  .loop-parse-settings {
    padding: var(--space-2);
    box-shadow: var(--el-box-shadow-light);
  }

  /* 单栏模式下主栏内的并排卡片（PC端两列、移动端单列） */
  .side-cards-inline {
    display: grid;
    grid-template-columns: 1fr;
    gap: var(--app-gap);
    align-items: stretch;
  }
  @media (min-width: 1025px) {
    .side-cards-inline {
      grid-template-columns: repeat(2, minmax(0, 1fr));
    }
    .side-cards-inline .help-section > .el-card,
    .side-cards-inline .author-card {
      height: 100%;
    }
  }
.option-line {
  display: inline-flex;
  align-items: center;
  gap: 8px;
}
</style>
