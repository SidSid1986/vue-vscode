<template>
  <div class="container" ref="containerRef">
    <!-- 左侧：文件树，固定 25% 宽度 -->
    <div class="file-content">
      <FileTree @fileSelected="fileSelected" />
    </div>

    <!-- 右侧：代码内容 + 拖拽 + Terminal，宽度 75% -->
    <div class="code-content" ref="codeContentRef">
      <!-- 上方：代码展示区，高度动态变化 -->
      <div class="editor-content" :style="{ height: editorHeight }">
        <!-- <pre>{{ strJson }}</pre> -->
        <MonacoCom
          ref="jsonComponents"
          :model-value="strJson"
          @update:model-value="handleChangeResponseJson"
        />
      </div>

      <!-- 中间：拖拽手柄 -->
      <div
        class="resize-handle"
        @mousedown="startDrag"
        :class="{ dragging: isDragging }"
      ></div>

      <!-- 下方：Terminal 组件，高度动态变化 -->
      <div class="terminal-container" :style="{ height: terminalHeight }">
        <Terminal :terminalHeight="terminalHeight" />
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, nextTick } from "vue";
import FileTree from "@/components/FileTree.vue";
import MonacoCom from "@/components/MonacoCom.vue";
import Terminal from "@/components/Terminal.vue";

// ========== 文件内容 ==========
const oneFileName = ref("");
const strJson = ref(
  "# Python 示例代码\nprint('Hello Python!')\nresult = 1 + 2\nprint('计算结果：', result)"
);

// ========== 高度相关 ==========
const codeContentRef = ref(null);
const editorHeight = ref(""); // 动态高度，单位 px
const terminalHeight = ref(""); // 动态高度，单位 px
const isDragging = ref(false);

// 拖拽相关变量
const startY = ref(0);
const startTerminalHeight = ref(0); // 拖拽起始时 terminal 高度（px）

// ========== 文件选择 ==========
const fileSelected = (file) => {
  console.log("File selected:", file);
  oneFileName.value = file.name;
  strJson.value = file.content;
};

// ========== 拖拽逻辑（你原来的逻辑，方向是对的！保留！） ==========

// 开始拖拽
const startDrag = (e) => {
  if (!codeContentRef.value) return;

  isDragging.value = true;

  const container = codeContentRef.value;
  const containerHeight = container.clientHeight; // 容器总高度（px）

  startY.value = e.clientY;
  startTerminalHeight.value = parseHeight(
    terminalHeight.value,
    containerHeight
  );

  document.addEventListener("mousemove", handleMouseMove);
  document.addEventListener("mouseup", stopDrag);

  e.preventDefault(); // 防止文本选中
};

// 拖拽中：计算新的 Terminal 和 Editor 高度
const handleMouseMove = (e) => {
  if (!isDragging.value) return;

  const container = codeContentRef.value;
  if (!container) return;

  const containerHeight = container.clientHeight;
  const deltaY = startY.value - e.clientY; // 注意：这里是 startY - e.clientY

  // newTerminalHeight = 初始 Terminal 高度 + deltaY
  // newEditorHeight = 初始总高度 - newTerminalHeight
  let newTerminalHeightPx = startTerminalHeight.value + deltaY;

  // 容器总高度
  const totalHeight = containerHeight;

  // 限制最小高度
  const minTerminalHeight = 100;
  const minEditorHeight = 100;

  newTerminalHeightPx = Math.max(
    minTerminalHeight,
    Math.min(newTerminalHeightPx, totalHeight - minEditorHeight)
  );

  const newEditorHeightPx = totalHeight - newTerminalHeightPx;

  // 设置高度（px）
  terminalHeight.value = `${newTerminalHeightPx}px`;
  editorHeight.value = `${newEditorHeightPx}px`;
};

// 停止拖拽
const stopDrag = () => {
  isDragging.value = false;
  document.removeEventListener("mousemove", handleMouseMove);
  document.removeEventListener("mouseup", stopDrag);
};

// 工具：将 "30vh" 或 "400px" 转为 px 数字
const parseHeight = (heightStr, containerHeight) => {
  if (heightStr.endsWith("px")) {
    return parseFloat(heightStr);
  } else if (heightStr.endsWith("vh")) {
    return (parseFloat(heightStr) / 100) * containerHeight;
  }
  return containerHeight / 2; // 默认值，备用
};

// ========== 生命周期 ==========
onMounted(() => {
  nextTick(() => {
    // 初始化高度（可选，比如默认各占一半）
    const containerHeight = codeContentRef.value?.clientHeight || 600;
    const initialTerminalHeight = containerHeight * 0.3; // 30%
    const initialEditorHeight = containerHeight - initialTerminalHeight;

    terminalHeight.value = `${initialTerminalHeight}px`;
    editorHeight.value = `${initialEditorHeight}px`;
  });
});

onUnmounted(() => {
  document.removeEventListener("mousemove", handleMouseMove);
  document.removeEventListener("mouseup", stopDrag);
});
</script>

<style scoped>
.container {
  display: flex;
  flex-direction: row;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  user-select: none;
  -webkit-user-select: none;
}

.file-content {
  width: 25vw;
  height: 100vh;
  border: 1px solid green;
}

.code-content {
  width: 75vw;
  height: 100vh;
  border: 1px solid red;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.editor-content {
  width: 100%;
  overflow: auto;
  box-sizing: border-box;
  padding: 10px;
  background: #f6f6f6;
}

.terminal-container {
  width: 100%;
  box-sizing: border-box;
  overflow: hidden;
  background-color: #1e1e1e; /* 模拟终端背景色 */
}

.resize-handle {
  height: 6px;
  background-color: #e0e0e0;
  cursor: ns-resize;
  user-select: none;
  border-top: 1px solid #ccc;
  border-bottom: 1px solid #ccc;
  transition: background-color 0.2s;
  flex-shrink: 0;
  position: relative;
  z-index: 10;
}

.resize-handle:hover {
  background-color: #d0d0d0;
}

.resize-handle.dragging {
  background-color: #a0a0a0;
  height: 8px;
}

/* 防止文本选择与拖拽冲突 */
* {
  -webkit-user-drag: none;
  touch-action: none;
}
</style>
