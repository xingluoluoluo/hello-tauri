<template>
  <div class="app-container">
    <!-- 装饰性背景元素 -->
    <div class="bg-pattern"></div>
    <div class="bg-gradient-orb"></div>

    <!-- 侧边栏 -->
    <aside class="sidebar">
      <div class="sidebar-header">
        <div class="logo-mark">
          <span class="logo-bar"></span>
          <span class="logo-bar"></span>
          <span class="logo-bar"></span>
        </div>
        <h2>应用管理</h2>
      </div>

      <nav class="sidebar-nav">
        <div class="nav-label">工作区</div>
        <button
          v-for="(tab, index) in tabs"
          :key="tab.id"
          :class="['tab-button', { active: activeTab === tab.id }]"
          @click="switchTab(tab.id)"
          :style="{ '--delay': `${index * 0.1}s` }"
        >
          <span class="tab-icon-wrapper">
            <span class="tab-icon">{{ tab.icon }}</span>
            <span class="icon-ring"></span>
          </span>
          <span class="tab-content">
            <span class="tab-label">{{ tab.name }}</span>
            <!-- <span class="tab-port">:{{ tab.port }}</span> -->
          </span>
          <span class="active-indicator"></span>
        </button>
      </nav>

      <div class="sidebar-footer">
        <div class="status-dot" :class="{ online: !loading }"></div>
        <span>{{ loading ? '同步中' : '已连接' }}</span>
      </div>
    </aside>

    <!-- 主内容区 -->
    <main class="main-content">
      <!-- 顶部操作栏 -->
      <header class="content-header">
        <div class="header-left">
          <span class="breadcrumb-prefix">工作区</span>
          <span class="breadcrumb-sep">/</span>
          <h1>{{ currentTab?.name || '加载中...' }}</h1>
        </div>
        <div class="header-right">
          <div v-if="loading" class="loading-indicator">
            <span class="loading-spinner"></span>
            <span>加载中</span>
          </div>
          <div v-else class="status-badge">
            <span class="status-icon">✓</span>
            <span>就绪</span>
          </div>
        </div>
      </header>

      <!-- iframe 内容区 -->
      <div class="iframe-container">
        <div v-if="!iframeSrc && !loading" class="placeholder">
          <div class="placeholder-content">
            <div class="placeholder-icon">
              <span></span>
              <span></span>
            </div>
            <p>等待资源加载</p>
            <span class="placeholder-hint">选择左侧应用开始</span>
          </div>
        </div>
        <div v-else-if="loading" class="loading-screen">
          <div class="loader">
            <div class="loader-bar"></div>
            <div class="loader-bar"></div>
            <div class="loader-bar"></div>
          </div>
          <p>正在获取资源...</p>
        </div>
        <iframe
          v-else-if="iframeSrc"
          :src="iframeSrc"
          class="content-iframe"
          allowfullscreen
        ></iframe>
      </div>
    </main>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from "vue";
import JSZip from "jszip";
import {
  exists,
  mkdir,
  writeFile,
  remove,
  readTextFile,
  BaseDirectory
} from "@tauri-apps/plugin-fs";
import { join, dirname, appDataDir } from "@tauri-apps/api/path";
import { convertFileSrc } from "@tauri-apps/api/core";

// 配置多个端口资源
const tabs = ref([
  { id: 'app1', name: '应用一', port: 8080, icon: '📦' },
  { id: 'app2', name: '应用二', port: 8081, icon: '🎯' },
]);

const activeTab = ref(null);
const iframeSrc = ref("");
const loading = ref(false);

const currentTab = computed(() => {
  return tabs.value.find(tab => tab.id === activeTab.value);
});

// 切换 tab 并加载资源
async function switchTab(tabId) {
  activeTab.value = tabId;
  await loadRemoteDist(tabId);
}

// 获取目标文件夹名
function getTargetFolder(port) {
  return `dynamic-dist-${port}`;
}

// 加载远程资源
async function loadRemoteDist(tabId) {
  const tab = tabs.value.find(t => t.id === tabId);
  if (!tab) return;

  const remoteUrl = `http://localhost:${tab.port}/dist.zip`;
  const TARGET_FOLDER = getTargetFolder(tab.port);

  if (loading.value) return;
  loading.value = true;
  iframeSrc.value = "";

  try {
    console.log(`正在从 ${remoteUrl} 下载...`);

    // 1. 下载 zip 文件
    const response = await fetch(remoteUrl);
    if (!response.ok)
      throw new Error(`下载失败: ${response.status} ${response.statusText}`);

    const zipBuffer = await response.arrayBuffer();

    // 2. 解压
    console.log("正在解压...");
    const zip = await JSZip.loadAsync(zipBuffer);

    // 3. 准备目标目录
    if (await exists(TARGET_FOLDER, { baseDir: BaseDirectory.AppData })) {
      console.log("清理旧目录...");
      await remove(TARGET_FOLDER, {
        baseDir: BaseDirectory.AppData,
        recursive: true,
      });
    }

    await mkdir(TARGET_FOLDER, {
      baseDir: BaseDirectory.AppData,
      recursive: true,
    });

    // 4. 逐个写入文件
    console.log("正在写入文件...");
    const writePromises = [];

    zip.forEach((relativePath, zipEntry) => {
      if (zipEntry.dir) return;

      let normalizedPath = relativePath;
      if (normalizedPath.startsWith("dist/")) {
        normalizedPath = normalizedPath.substring(5);
      }

      writePromises.push(
        (async () => {
          const filePath = await join(TARGET_FOLDER, normalizedPath);

          const parentDir = await dirname(filePath);
          if (parentDir && parentDir !== ".") {
            await mkdir(parentDir, {
              baseDir: BaseDirectory.AppData,
              recursive: true,
            });
          }

          const content = await zipEntry.async("uint8array");
          await writeFile(filePath, content, {
            baseDir: BaseDirectory.AppData,
          });
        })()
      );
    });

    await Promise.all(writePromises);

    console.log("文件写入完成，开始处理 index.html...");

    const appData = await appDataDir();
    const indexRelativePath = await join(TARGET_FOLDER, "index.html");
    const indexFullPath = await join(appData, TARGET_FOLDER, "index.html");

    const indexExists = await exists(indexRelativePath, { baseDir: BaseDirectory.AppData });
    console.log("index.html 是否存在:", indexExists);

    if (!indexExists) {
      throw new Error("index.html 不存在，请检查 zip 包结构");
    }

    // 读取并修改 index.html
    let indexContent = await readTextFile(indexRelativePath, { baseDir: BaseDirectory.AppData });

    const baseDirPath = convertFileSrc(await join(appData, TARGET_FOLDER));
    const baseHref = baseDirPath.endsWith("/") ? baseDirPath : `${baseDirPath}/`;

    if (/<\s*base\b/i.test(indexContent)) {
      indexContent = indexContent.replace(
        /<\s*base[^>]*>/i,
        `<base href="${baseHref}">`
      );
    } else {
      indexContent = indexContent.replace(
        /<\s*head[^>]*>/i,
        (match) => `${match}\n  <base href="${baseHref}">`
      );
    }

    indexContent = indexContent.replace(
      /(src|href|action)=["']\/(assets\/[^"']+)["']/gi,
      (match, attr, path) => `${attr}="./${path}"`
    );

    indexContent = indexContent.replace(
      /(src|href|action)=["']\/(?!\/)([^"']+)["']/gi,
      (match, attr, path) => `${attr}="./${path}"`
    );

    await writeFile(
      indexRelativePath,
      new TextEncoder().encode(indexContent),
      { baseDir: BaseDirectory.AppData }
    );

    console.log("已注入 <base> 并修复资源路径，baseHref =", baseHref);

    // 设置 iframe src
    iframeSrc.value = convertFileSrc(indexFullPath);

    console.log("远程 dist 加载成功！iframe src:", iframeSrc.value);
  } catch (err) {
    console.error("加载远程 dist 失败:", err);
    alert(`加载失败: ${err.message || err}`);
  } finally {
    loading.value = false;
  }
}

// 初始化时默认选中第一个并加载
onMounted(() => {
  if (tabs.value.length > 0) {
    switchTab(tabs.value[0].id);
  }
});
</script>

<style>
@import url('https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@400;500;700&family=DM+Sans:wght@400;500;600&display=swap');

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

:root {
  --bg-dark: #0a0a0b;
  --bg-card: #141416;
  --bg-elevated: #1a1a1d;
  --accent: #00f5d4;
  --accent-dim: rgba(0, 245, 212, 0.15);
  --accent-glow: rgba(0, 245, 212, 0.4);
  --text-primary: #ffffff;
  --text-secondary: #8a8a8e;
  --text-muted: #5a5a5e;
  --border-subtle: rgba(255, 255, 255, 0.06);
  --border-strong: rgba(255, 255, 255, 0.12);
  --content-bg: #fafaf9;
  --content-card: #ffffff;
  --shadow-soft: 0 8px 32px rgba(0, 0, 0, 0.4);
  --radius-sm: 4px;
  --radius-md: 8px;
  --radius-lg: 16px;
}

.app-container {
  display: flex;
  height: 100vh;
  font-family: 'DM Sans', sans-serif;
  background: var(--bg-dark);
  position: relative;
  overflow: hidden;
}

/* 装饰性背景 */
.bg-pattern {
  position: fixed;
  inset: 0;
  background-image:
    linear-gradient(var(--border-subtle) 1px, transparent 1px),
    linear-gradient(90deg, var(--border-subtle) 1px, transparent 1px);
  background-size: 60px 60px;
  opacity: 0.5;
  pointer-events: none;
  z-index: 0;
}

.bg-gradient-orb {
  position: fixed;
  width: 600px;
  height: 600px;
  background: radial-gradient(circle, var(--accent-glow) 0%, transparent 70%);
  top: -200px;
  right: -200px;
  opacity: 0.3;
  pointer-events: none;
  z-index: 0;
  filter: blur(80px);
}

/* 侧边栏样式 */
.sidebar {
  width: 260px;
  background: var(--bg-card);
  color: var(--text-primary);
  display: flex;
  flex-direction: column;
  position: relative;
  z-index: 10;
  border-right: 1px solid var(--border-subtle);
}

.sidebar::before {
  content: '';
  position: absolute;
  top: 0;
  right: 0;
  width: 1px;
  height: 100%;
  background: linear-gradient(
    to bottom,
    transparent,
    var(--accent) 30%,
    var(--accent) 70%,
    transparent
  );
  opacity: 0.5;
}

.sidebar-header {
  padding: 24px 20px;
  border-bottom: 1px solid var(--border-subtle);
  display: flex;
  align-items: center;
  gap: 12px;
}

.logo-mark {
  width: 36px;
  height: 36px;
  background: var(--accent-dim);
  border-radius: var(--radius-md);
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  gap: 3px;
  padding: 8px;
}

.logo-bar {
  width: 100%;
  height: 2px;
  background: var(--accent);
  border-radius: 1px;
}

.logo-bar:nth-child(2) {
  width: 70%;
}

.sidebar-header h2 {
  font-family: 'Space Grotesk', sans-serif;
  font-size: 16px;
  font-weight: 700;
  letter-spacing: -0.02em;
  flex: 1;
}

.version-tag {
  font-size: 10px;
  font-weight: 600;
  color: var(--accent);
  background: var(--accent-dim);
  padding: 3px 8px;
  border-radius: 100px;
  letter-spacing: 0.05em;
}

.sidebar-nav {
  flex: 1;
  padding: 16px 12px;
  overflow-y: auto;
}

.nav-label {
  font-size: 11px;
  font-weight: 600;
  color: var(--text-muted);
  text-transform: uppercase;
  letter-spacing: 0.1em;
  padding: 8px 12px;
  margin-bottom: 8px;
}

.tab-button {
  width: 100%;
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 14px 16px;
  margin-bottom: 6px;
  background: transparent;
  border: 1px solid transparent;
  border-radius: var(--radius-md);
  color: var(--text-secondary);
  cursor: pointer;
  transition: all 0.25s cubic-bezier(0.4, 0, 0.2, 1);
  font-size: 14px;
  position: relative;
  overflow: hidden;
  animation: fadeSlideIn 0.4s ease-out backwards;
  animation-delay: var(--delay);
}

@keyframes fadeSlideIn {
  from {
    opacity: 0;
    transform: translateX(-10px);
  }
  to {
    opacity: 1;
    transform: translateX(0);
  }
}

.tab-button::before {
  content: '';
  position: absolute;
  inset: 0;
  background: linear-gradient(135deg, var(--accent-dim) 0%, transparent 50%);
  opacity: 0;
  transition: opacity 0.25s ease;
}

.tab-button:hover {
  background: var(--bg-elevated);
  color: var(--text-primary);
  border-color: var(--border-subtle);
}

.tab-button:hover::before {
  opacity: 1;
}

.tab-button.active {
  background: var(--accent-dim);
  color: var(--accent);
  border-color: var(--accent);
}

.tab-button.active::before {
  opacity: 1;
}

.active-indicator {
  position: absolute;
  right: 0;
  top: 50%;
  transform: translateY(-50%) scaleY(0);
  width: 3px;
  height: 60%;
  background: var(--accent);
  border-radius: 3px 0 0 3px;
  transition: transform 0.25s cubic-bezier(0.4, 0, 0.2, 1);
}

.tab-button.active .active-indicator {
  transform: translateY(-50%) scaleY(1);
}

.tab-icon-wrapper {
  width: 36px;
  height: 36px;
  background: var(--bg-dark);
  border-radius: var(--radius-md);
  display: flex;
  align-items: center;
  justify-content: center;
  position: relative;
  transition: all 0.25s ease;
}

.tab-button.active .tab-icon-wrapper {
  background: var(--accent);
}

.tab-icon {
  font-size: 16px;
  z-index: 1;
  position: relative;
}

.icon-ring {
  position: absolute;
  inset: -2px;
  border: 2px solid var(--accent);
  border-radius: var(--radius-md);
  opacity: 0;
  transform: scale(1.2);
  transition: all 0.25s ease;
}

.tab-button.active .icon-ring {
  opacity: 0.3;
  transform: scale(1);
}

.tab-content {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: 2px;
  position: relative;
  z-index: 1;
}

.tab-label {
  font-weight: 600;
  font-size: 13px;
}

.tab-port {
  font-size: 11px;
  font-family: 'Space Grotesk', monospace;
  color: var(--text-muted);
  opacity: 0.7;
}

.tab-button.active .tab-port {
  color: var(--accent);
  opacity: 0.8;
}

.sidebar-footer {
  padding: 16px 20px;
  border-top: 1px solid var(--border-subtle);
  display: flex;
  align-items: center;
  gap: 10px;
  font-size: 12px;
  color: var(--text-secondary);
}

.status-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: var(--text-muted);
  transition: all 0.3s ease;
}

.status-dot.online {
  background: var(--accent);
  box-shadow: 0 0 12px var(--accent-glow);
}

/* 主内容区样式 */
.main-content {
  flex: 1;
  display: flex;
  flex-direction: column;
  background: var(--content-bg);
  position: relative;
  z-index: 5;
}

.content-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 20px 32px;
  background: var(--content-card);
  border-bottom: 1px solid rgba(0, 0, 0, 0.06);
  position: relative;
}

.content-header::after {
  content: '';
  position: absolute;
  bottom: 0;
  left: 32px;
  right: 32px;
  height: 1px;
  background: linear-gradient(
    90deg,
    transparent,
    var(--accent) 50%,
    transparent
  );
  opacity: 0.3;
}

.header-left {
  display: flex;
  align-items: center;
  gap: 8px;
}

.breadcrumb-prefix {
  font-size: 13px;
  color: var(--text-muted);
  font-weight: 500;
}

.breadcrumb-sep {
  color: #d0d0d0;
  font-size: 12px;
}

.content-header h1 {
  font-family: 'Space Grotesk', sans-serif;
  font-size: 22px;
  font-weight: 700;
  color: #1a1a1a;
  letter-spacing: -0.02em;
}

.header-right {
  display: flex;
  align-items: center;
  gap: 12px;
}

.loading-indicator {
  display: flex;
  align-items: center;
  gap: 10px;
  font-size: 13px;
  color: var(--text-secondary);
  background: var(--bg-card);
  padding: 8px 16px;
  border-radius: 100px;
}

.loading-spinner {
  width: 14px;
  height: 14px;
  border: 2px solid var(--border-strong);
  border-top-color: var(--accent);
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.status-badge {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 13px;
  font-weight: 600;
  color: #0a9b8a;
  background: rgba(0, 245, 212, 0.1);
  padding: 8px 16px;
  border-radius: 100px;
}

.status-icon {
  font-size: 12px;
}

/* iframe 容器 */
.iframe-container {
  flex: 1;
  padding: 24px;
  overflow: hidden;
  display: flex;
}

.placeholder {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  background: var(--content-card);
  border-radius: var(--radius-lg);
  border: 2px dashed rgba(0, 0, 0, 0.08);
}

.placeholder-content {
  text-align: center;
}

.placeholder-icon {
  width: 64px;
  height: 64px;
  margin: 0 auto 16px;
  background: linear-gradient(135deg, #f0f0f0 0%, #e8e8e8 100%);
  border-radius: var(--radius-md);
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  gap: 6px;
  padding: 16px;
}

.placeholder-icon span {
  width: 100%;
  height: 4px;
  background: #ccc;
  border-radius: 2px;
}

.placeholder-icon span:last-child {
  width: 60%;
}

.placeholder p {
  font-size: 15px;
  font-weight: 600;
  color: #666;
  margin-bottom: 6px;
}

.placeholder-hint {
  font-size: 13px;
  color: #999;
}

.loading-screen {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  background: var(--content-card);
  border-radius: var(--radius-lg);
}

.loader {
  display: flex;
  gap: 6px;
  margin-bottom: 20px;
}

.loader-bar {
  width: 6px;
  height: 32px;
  background: linear-gradient(to top, var(--accent), #00d4aa);
  border-radius: 3px;
  animation: loaderPulse 1s ease-in-out infinite;
}

.loader-bar:nth-child(2) {
  animation-delay: 0.15s;
}

.loader-bar:nth-child(3) {
  animation-delay: 0.3s;
}

@keyframes loaderPulse {
  0%, 100% {
    transform: scaleY(0.4);
    opacity: 0.5;
  }
  50% {
    transform: scaleY(1);
    opacity: 1;
  }
}

.loading-screen p {
  font-size: 14px;
  color: #888;
  font-weight: 500;
}

.content-iframe {
  flex: 1;
  width: 100%;
  height: 100%;
  border: none;
  border-radius: var(--radius-lg);
  background: var(--content-card);
  box-shadow:
    0 4px 24px rgba(0, 0, 0, 0.06),
    0 0 0 1px rgba(0, 0, 0, 0.04);
  animation: iframeReveal 0.5s ease-out;
}

@keyframes iframeReveal {
  from {
    opacity: 0;
    transform: scale(0.98);
  }
  to {
    opacity: 1;
    transform: scale(1);
  }
}

/* 滚动条美化 */
::-webkit-scrollbar {
  width: 6px;
}

::-webkit-scrollbar-track {
  background: transparent;
}

::-webkit-scrollbar-thumb {
  background: var(--border-strong);
  border-radius: 3px;
}

::-webkit-scrollbar-thumb:hover {
  background: var(--text-muted);
}
</style>
