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
        <div class="status-dot" :class="{ online: !initializing && !loading }"></div>
        <span>{{ initializing ? '初始化中' : (loading ? '同步中' : '已连接') }}</span>
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
          <div v-if="initializing" class="loading-indicator">
            <span class="loading-spinner"></span>
            <span>初始化资源</span>
          </div>
          <div v-else-if="loading" class="loading-indicator">
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
        <div v-if="initializing" class="loading-screen">
          <div class="loader">
            <div class="loader-bar"></div>
            <div class="loader-bar"></div>
            <div class="loader-bar"></div>
          </div>
          <p>正在初始化资源...</p>
        </div>
        <div v-else-if="!iframeSrc" class="placeholder">
          <div class="placeholder-content">
            <div class="placeholder-icon">
              <span></span>
              <span></span>
            </div>
            <p>等待资源加载</p>
            <span class="placeholder-hint">选择左侧应用开始</span>
          </div>
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
  readDir,
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
const initializing = ref(true);

// 缓存每个 tab 的 iframeSrc
const tabResourceCache = new Map();

const currentTab = computed(() => {
  return tabs.value.find(tab => tab.id === activeTab.value);
});

// 切换 tab（使用缓存，不重新请求远程资源）
function switchTab(tabId) {
  activeTab.value = tabId;

  // 直接从缓存读取，不请求远程
  if (tabResourceCache.has(tabId)) {
    iframeSrc.value = tabResourceCache.get(tabId);
  }
}

// 获取目标文件夹名
function getTargetFolder(port) {
  return `dynamic-dist-${port}`;
}

// 计算内容的 hash
async function computeHash(buffer) {
  const hashBuffer = await crypto.subtle.digest('SHA-256', buffer);
  return Array.from(new Uint8Array(hashBuffer)).map(b => b.toString(16).padStart(2, '0')).join('');
}

// 获取目录下所有文件的相对路径
async function listAllFiles(folder) {
  const files = new Set();

  async function scan(dir, prefix = '') {
    try {
      const entries = await readDir(dir, { baseDir: BaseDirectory.AppData });
      for (const entry of entries) {
        const path = prefix ? `${prefix}/${entry.name}` : entry.name;
        if (entry.children) {
          await scan(await join(folder, path), path);
        } else {
          files.add(path);
        }
      }
    } catch (e) {
      // 目录不存在或读取失败
    }
  }

  await scan(folder);
  return files;
}

// 加载远程资源（增量更新），返回 iframeSrc
async function loadRemoteDist(tabId) {
  const tab = tabs.value.find(t => t.id === tabId);
  if (!tab) return "";

  const remoteUrl = `http://localhost:${tab.port}/dist.zip`;
  const TARGET_FOLDER = getTargetFolder(tab.port);

  try {
    console.log(`[${tab.name}] 正在从 ${remoteUrl} 下载...`);

    // 1. 下载 zip 文件
    const response = await fetch(remoteUrl);
    if (!response.ok)
      throw new Error(`下载失败: ${response.status} ${response.statusText}`);

    const zipBuffer = await response.arrayBuffer();

    // 2. 解压
    console.log(`[${tab.name}] 正在解压...`);
    const zip = await JSZip.loadAsync(zipBuffer);

    // 3. 收集 zip 中的文件信息
    const zipFilePaths = new Set();

    zip.forEach((relativePath, zipEntry) => {
      if (zipEntry.dir) return;

      let normalizedPath = relativePath;
      if (normalizedPath.startsWith("dist/")) {
        normalizedPath = normalizedPath.substring(5);
      }

      zipFilePaths.add(normalizedPath);
    });

    // 4. 检查目标目录是否存在
    const targetExists = await exists(TARGET_FOLDER, { baseDir: BaseDirectory.AppData });
    let needsFullWrite = !targetExists;

    // 5. 如果目录存在，检查 index.html 和 assets 文件是否有变化
    if (targetExists) {
      console.log(`[${tab.name}] 检测到已有目录，进行增量对比...`);

      // 读取现有的 index.html 内容
      const indexRelativePath = await join(TARGET_FOLDER, "index.html");
      let existingIndexContent = null;
      try {
        existingIndexContent = await readTextFile(indexRelativePath, { baseDir: BaseDirectory.AppData });
      } catch (e) {
        console.log(`[${tab.name}] 无法读取现有 index.html，需要完整更新`);
        needsFullWrite = true;
      }

      // 从 zip 中获取新的 index.html 内容（未修改的原始内容）
      const zipIndexEntry = zip.file("dist/index.html") || zip.file("index.html");
      if (!zipIndexEntry) {
        throw new Error("zip 包中找不到 index.html");
      }
      const newIndexContent = await zipIndexEntry.async("text");

      // 对比 index.html 原始内容
      if (!needsFullWrite && existingIndexContent) {
        // 从现有 index.html 中提取原始内容（移除 base href 和路径修改）
        let existingOriginal = existingIndexContent
          .replace(/<base href="[^"]*">\n?/i, '')
          .replace(/\.\//g, '/');

        // 计算内容 hash 对比
        const existingHash = await computeHash(new TextEncoder().encode(existingOriginal));
        const newHash = await computeHash(new TextEncoder().encode(newIndexContent));

        if (existingHash !== newHash) {
          console.log(`[${tab.name}] index.html 内容有变化，需要更新`);
          needsFullWrite = true;
        }
      }

      // 检查 assets 文件是否变化（通过文件名对比）
      if (!needsFullWrite) {
        const existingFiles = await listAllFiles(TARGET_FOLDER);

        // 检查 assets 目录下的文件
        const existingAssets = new Set();
        const newAssets = new Set();

        for (const file of existingFiles) {
          if (file.startsWith('assets/')) {
            existingAssets.add(file);
          }
        }

        for (const file of zipFilePaths) {
          if (file.startsWith('assets/')) {
            newAssets.add(file);
          }
        }

        // 对比 assets 文件集合
        const assetsChanged = existingAssets.size !== newAssets.size ||
          [...existingAssets].some(f => !newAssets.has(f)) ||
          [...newAssets].some(f => !existingAssets.has(f));

        if (assetsChanged) {
          console.log(`[${tab.name}] assets 文件有变化，需要更新`);
          needsFullWrite = true;
        }
      }
    }

    // 6. 根据对比结果决定更新策略
    if (needsFullWrite) {
      console.log(`[${tab.name}] 执行增量更新...`);

      // 确保目录存在
      await mkdir(TARGET_FOLDER, {
        baseDir: BaseDirectory.AppData,
        recursive: true,
      });

      // 获取现有文件列表
      const existingFiles = targetExists ? await listAllFiles(TARGET_FOLDER) : new Set();

      // 找出需要删除的文件（在新 zip 中不存在）
      const filesToDelete = [...existingFiles].filter(f => !zipFilePaths.has(f));

      // 写入文件（只写入需要更新的）
      console.log(`[${tab.name}] 正在写入文件...`);
      const writePromises = [];

      for (const normalizedPath of zipFilePaths) {
        const zipEntry = zip.file(`dist/${normalizedPath}`) || zip.file(normalizedPath);
        if (!zipEntry) continue;

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
      }

      await Promise.all(writePromises);

      // 删除不再需要的文件
      for (const file of filesToDelete) {
        try {
          await remove(await join(TARGET_FOLDER, file), { baseDir: BaseDirectory.AppData });
          console.log(`[${tab.name}] 删除旧文件: ${file}`);
        } catch (e) {
          console.warn(`[${tab.name}] 删除文件失败: ${file}`, e);
        }
      }
    } else {
      console.log(`[${tab.name}] 文件无变化，跳过文件写入`);
    }

    console.log(`[${tab.name}] 开始处理 index.html...`);

    const appData = await appDataDir();
    const indexRelativePath = await join(TARGET_FOLDER, "index.html");
    const indexFullPath = await join(appData, TARGET_FOLDER, "index.html");

    const indexExists = await exists(indexRelativePath, { baseDir: BaseDirectory.AppData });
    console.log(`[${tab.name}] index.html 是否存在:`, indexExists);

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

    console.log(`[${tab.name}] 已注入 <base> 并修复资源路径`);

    // 返回 iframe src
    const src = convertFileSrc(indexFullPath);
    console.log(`[${tab.name}] 远程 dist 加载成功！`);
    return src;
  } catch (err) {
    console.error(`[${tab.name}] 加载远程 dist 失败:`, err);
    throw err;
  }
}

// 初始化时加载所有远程资源
async function initializeResources() {
  initializing.value = true;
  loading.value = true;

  try {
    // 并行加载所有 tab 的资源
    await Promise.all(
      tabs.value.map(async (tab) => {
        const src = await loadRemoteDist(tab.id);
        tabResourceCache.set(tab.id, src);
      })
    );

    // 设置默认选中的 tab
    if (tabs.value.length > 0) {
      activeTab.value = tabs.value[0].id;
      iframeSrc.value = tabResourceCache.get(tabs.value[0].id);
    }
  } catch (err) {
    console.error("初始化资源失败:", err);
    alert(`初始化失败: ${err.message || err}`);
  } finally {
    initializing.value = false;
    loading.value = false;
  }
}

// 初始化
onMounted(() => {
  initializeResources();
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
