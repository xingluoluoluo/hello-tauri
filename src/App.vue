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
  readTextFile,
  BaseDirectory
} from "@tauri-apps/plugin-fs";
import { join, dirname, appDataDir } from "@tauri-apps/api/path";
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

    // 4. 逐个写入文件（支持去掉顶层 dist/ 前缀）
    console.log("正在写入文件...");
    const writePromises = [];

    zip.forEach((relativePath, zipEntry) => {
      if (zipEntry.dir) return; // 跳过目录

      // 去掉可能的顶层 dist/ 目录
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

    // 读取原始内容
    let indexContent = await readTextFile(indexRelativePath, { baseDir: BaseDirectory.AppData });

    // 生成正确的 base href（必须以 / 结尾）
    const baseDirPath = convertFileSrc(await join(appData, TARGET_FOLDER));
    const baseHref = baseDirPath.endsWith("/") ? baseDirPath : `${baseDirPath}/`;

    // 1. 注入或更新 <base> 标签
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

    // 2. 关键修复：把所有以 /assets/ 开头的路径改成 ./assets/ （去掉开头的 /）
    // 这会让 JS、CSS、图片等资源受 <base> 控制
    indexContent = indexContent.replace(
      /(src|href|action)=["']\/(assets\/[^"']+)["']/gi,
      (match, attr, path) => `${attr}="./${path}"`
    );

    // 额外处理可能存在的其他根相对路径（如 /img/、/static/ 等，根据你的 dist 情况调整）
    indexContent = indexContent.replace(
      /(src|href|action)=["']\/(?!\/)([^"']+)["']/gi,
      (match, attr, path) => `${attr}="./${path}"`
    );

    // 写回修改后的 index.html
    await writeFile(
      indexRelativePath,
      new TextEncoder().encode(indexContent),
      { baseDir: BaseDirectory.AppData }
    );

    console.log("已注入 <base> 并修复资源路径，baseHref =", baseHref);
    console.log("处理后的 index.html head 部分预览：");
    console.log(indexContent.match(/<head[^>]*>[\s\S]*?<\/head>/i)?.[0] || "未找到 head");

    // 6. 设置 iframe src
    iframeSrc.value = convertFileSrc(indexFullPath);

    console.log("远程 dist 加载成功！iframe src 已设置：", iframeSrc.value);
  } catch (err) {
    console.error("加载远程 dist 失败:", err);
    alert(`加载失败: ${err.message || err}`);
  } finally {
    loading.value = false;
  }
}
</script>