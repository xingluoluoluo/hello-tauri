<template>
  <div class="dynamic-embed">
    <button @click="loadRemoteDist" :disabled="loading">
      {{ loading ? "加载中..." : "从远程加载 dist 并显示" }}
    </button>

    <!-- 动态 iframe -->
    <iframe
      v-if="iframeSrc"
      :src="iframeSrc"
      style="width: 100%; height: 100vh; border: none; display: block"
      allowfullscreen
    ></iframe>
  </div>
</template>

<script setup>
import { ref } from "vue";
import JSZip from "jszip";
import {
  exists,
  mkdir,
  writeFile,
  remove,
  BaseDirectory,
} from "@tauri-apps/plugin-fs";
import { join } from "@tauri-apps/api/path";
import { convertFileSrc } from "@tauri-apps/api/core";

const iframeSrc = ref("");
const loading = ref(false);

const TARGET_FOLDER = "dynamic-dist"; // 目标文件夹名（相对 AppData）

async function loadRemoteDist() {
  const remoteUrl = "http://localhost:8080/dist.zip"; // ← 改成你的远程 dist zip 地址

  if (loading.value) return;
  loading.value = true;
  iframeSrc.value = ""; // 清空旧内容

  try {
    console.log("正在下载远程 dist...");

    // 1. 下载 zip 文件
    const response = await fetch(remoteUrl);
    if (!response.ok)
      throw new Error(`下载失败: ${response.status} ${response.statusText}`);

    const zipBuffer = await response.arrayBuffer();

    // 2. 解压
    console.log("正在解压...");
    const zip = await JSZip.loadAsync(zipBuffer);

    // 3. 准备目标目录（使用 BaseDirectory.AppData）
    if (await exists(TARGET_FOLDER, { baseDir: BaseDirectory.AppData })) {
      console.log("清理旧目录...");
      await remove(TARGET_FOLDER, {
        baseDir: BaseDirectory.AppData,
        recursive: true,
      });
    }

    // 创建目标目录（recursive: true 自动创建父目录）
    await mkdir(TARGET_FOLDER, {
      baseDir: BaseDirectory.AppData,
      recursive: true,
    });

    // 4. 逐个写入文件
    console.log("正在写入文件...");
    const writePromises = [];

    zip.forEach((relativePath, zipEntry) => {
      if (zipEntry.dir) return; // 跳过目录

      writePromises.push(
        (async () => {
          const filePath = await join(TARGET_FOLDER, relativePath);

          // 确保父目录存在
          const parentDir = filePath.substring(0, filePath.lastIndexOf("/"));
          if (parentDir) {
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

    // 5. 生成 iframe 可安全加载的 URL
    const indexHtmlPath = await join(TARGET_FOLDER, "index.html");
    iframeSrc.value = convertFileSrc(indexHtmlPath);

    console.log("远程 dist 加载成功！iframe src 已设置");
  } catch (err) {
    console.error("加载远程 dist 失败:", err);
    alert(`加载失败: ${err.message || err}`);
  } finally {
    loading.value = false;
  }
}
</script>