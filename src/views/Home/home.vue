<template>
  <div class="container" ref="containerRef">
    <div class="file-content">
      <FileTree @fileSelected="fileSelected" />
    </div>

    <div class="code-content" ref="codeContentRef">
      <!-- 代码编辑区域 -->
      <div class="editor-content" :style="{ height: editorHeight }">
        <pre>{{ strJson }}</pre>
      </div>

      <!-- 拖拽手柄 -->
      <div
        class="resize-handle"
        @mousedown="startDrag"
        :class="{ dragging: isDragging }"
      ></div>

      <!-- 终端区域 -->
      <div class="terminal-container" :style="{ height: terminalHeight }">
        <!-- <Terminal :terminalHeight="terminalHeight" /> -->
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, nextTick } from "vue";
import FileTree from "@/components/FileTree.vue";
import Terminal from "@/components/Terminal.vue";

const oneFileName = ref("");
const strJson = ref(
  "# Python 示例代码\nprint('Hello Python!')\nresult = 1 + 2\nprint('计算结果：', result)"
);

// 高度相关变量
const terminalHeight = ref("30vh"); // 初始高度设为30%
const editorHeight = ref("");
const isDragging = ref(false);
const codeContentRef = ref(null);
const minTerminalHeight = 100; // 最小高度100px
const maxTerminalHeight = ref(0);

// 文件选择处理
const fileSelected = (file) => {
  console.log(file);
  oneFileName.value = file.name;
  strJson.value = file.content;
};

// 开始拖拽
const startDrag = (e) => {
  if (!codeContentRef.value) return;

  isDragging.value = true;
  e.preventDefault();

  // 获取容器总高度
  const containerHeight = codeContentRef.value.offsetHeight;
  maxTerminalHeight.value = containerHeight - minTerminalHeight;

  // 记录初始位置和高度
  const startY = e.clientY;
  const startTerminalHeight = parseHeight(
    terminalHeight.value,
    containerHeight
  );

  // 鼠标移动事件处理
  const handleMouseMove = (e) => {
    if (!isDragging.value) return;

    // 计算移动距离 - 向上拖拽增加高度，向下拖拽减小高度
    const deltaY = startY - e.clientY;
    let newHeight = startTerminalHeight + deltaY;

    // 限制在最小和最大高度之间
    newHeight = Math.max(
      minTerminalHeight,
      Math.min(maxTerminalHeight.value, newHeight)
    );

    // 更新高度
    updateHeights(newHeight, containerHeight);
  };

  // 鼠标释放事件处理
  const handleMouseUp = () => {
    isDragging.value = false;
    document.removeEventListener("mousemove", handleMouseMove);
    document.removeEventListener("mouseup", handleMouseUp);
  };

  // 添加全局事件监听
  document.addEventListener("mousemove", handleMouseMove);
  document.addEventListener("mouseup", handleMouseUp);
};

// 解析高度值
const parseHeight = (heightStr, containerHeight) => {
  if (heightStr.endsWith("px")) {
    return parseFloat(heightStr);
  } else if (heightStr.endsWith("vh")) {
    // 转换vh为px（基于容器高度而非视口）
    return (parseFloat(heightStr) / 100) * containerHeight;
  }
  return 0;
};

// 更新高度
const updateHeights = (terminalH, containerHeight) => {
  terminalHeight.value = `${terminalH}px`;
  editorHeight.value = `${containerHeight - terminalH}px`;
};

// 窗口大小变化时重新计算高度
const handleResize = () => {
  if (codeContentRef.value) {
    const containerHeight = codeContentRef.value.offsetHeight;
    const terminalH = parseHeight(terminalHeight.value, containerHeight);
    updateHeights(terminalH, containerHeight);
  }
};

onMounted(() => {
  // 初始化高度
  nextTick(() => {
    handleResize();
  });

  // 监听窗口大小变化
  window.addEventListener("resize", handleResize);
});

onUnmounted(() => {
  window.removeEventListener("resize", handleResize);
});
</script>

<style scoped>
.container {
  display: flex;
  flex-direction: row;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
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

  /* overflow: hidden; */
}

.editor-content {
  width: 100%;
  overflow: auto;
  box-sizing: border-box;
  padding: 10px;
  transition: height 0.1s ease-out;
}

.terminal-container {
  border: 2px solid pink;
  width: 100%;
  box-sizing: border-box;
  transition: height 0.1s ease-out;
  overflow: hidden;
  background-color: pink;
}

/* 拖拽手柄样式 */
.resize-handle {
  height: 6px;
  background-color: #e0e0e0;
  cursor: ns-resize;
  user-select: none;
  border-top: 1px solid #ccc;
  border-bottom: 1px solid #ccc;
  transition: background-color 0.2s;
}

.resize-handle:hover {
  background-color: #d0d0d0;
}

.resize-handle.dragging {
  background-color: #b0b0b0;
  height: 8px;
}
</style>
