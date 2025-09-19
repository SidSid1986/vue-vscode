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

// 拖拽起始信息
const startY = ref(0);
const startTerminalHeight = ref(0);

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
  const container = codeContentRef.value;
  const containerHeight = container.offsetHeight;

  // 记录初始状态（参考横向拖拽的简洁写法）
  startY.value = e.clientY;
  startTerminalHeight.value = parseHeight(
    terminalHeight.value,
    containerHeight
  );

  // 计算最大高度限制
  maxTerminalHeight.value = containerHeight - minTerminalHeight;

  // 添加全局事件监听（使用命名函数便于移除）
  document.addEventListener("mousemove", handleMouseMove);
  document.addEventListener("mouseup", stopDrag);

  // 阻止默认行为（防止文本选择）
  e.preventDefault();
};

// 拖拽中（核心优化点）
const handleMouseMove = (e) => {
  if (!isDragging.value) return;

  const container = codeContentRef.value;
  if (!container) return;

  const containerHeight = container.offsetHeight;
  const containerRect = container.getBoundingClientRect();

  // 计算鼠标在容器内的相对位置（参考横向拖拽的容器内坐标计算）
  const mouseY = e.clientY - containerRect.top;

  // 基于容器高度的比例计算新高度（更精准的计算方式）
  let newHeight = (mouseY / containerHeight) * containerHeight;
  newHeight = startTerminalHeight.value + (startY.value - e.clientY);

  // 边界限制（参考横向拖拽的简洁判断）
  newHeight = Math.max(
    minTerminalHeight,
    Math.min(maxTerminalHeight.value, newHeight)
  );

  // 直接更新高度，去掉中间变量
  terminalHeight.value = `${newHeight}px`;
  editorHeight.value = `${containerHeight - newHeight}px`;
};

// 停止拖拽（提取为独立函数，更清晰）
const stopDrag = () => {
  isDragging.value = false;
  document.removeEventListener("mousemove", handleMouseMove);
  document.removeEventListener("mouseup", stopDrag);
};

// 鼠标离开窗口时停止拖拽（新增，参考横向拖拽的鲁棒性处理）
const handleMouseLeave = () => {
  if (isDragging.value) {
    stopDrag();
  }
};

// 解析高度值
const parseHeight = (heightStr, containerHeight) => {
  if (heightStr.endsWith("px")) {
    return parseFloat(heightStr);
  } else if (heightStr.endsWith("vh")) {
    return (parseFloat(heightStr) / 100) * containerHeight;
  }
  return 0;
};

// 窗口大小变化时重新计算高度
const handleResize = () => {
  if (codeContentRef.value) {
    const containerHeight = codeContentRef.value.offsetHeight;
    const terminalH = parseHeight(terminalHeight.value, containerHeight);
    const newHeight = Math.min(terminalH, containerHeight - minTerminalHeight);
    terminalHeight.value = `${newHeight}px`;
    editorHeight.value = `${containerHeight - newHeight}px`;
  }
};

onMounted(() => {
  // 初始化高度
  nextTick(() => {
    handleResize();
  });

  // 监听窗口大小变化和鼠标离开事件
  window.addEventListener("resize", handleResize);
  document.addEventListener("mouseleave", handleMouseLeave);
});

onUnmounted(() => {
  window.removeEventListener("resize", handleResize);
  document.removeEventListener("mouseleave", handleMouseLeave);
  // 确保移除所有事件监听
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
  user-select: none; /* 新增：全局禁止文本选择 */
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
  overflow: hidden; /* 修复溢出问题 */
}

.editor-content {
  width: 100%;
  overflow: auto;
  box-sizing: border-box;
  padding: 10px;
  /* 优化过渡效果：更短的时间和更自然的曲线 */
  transition: height 0.08s ease-out;
}

.terminal-container {
  border: 2px solid pink;
  width: 100%;
  box-sizing: border-box;
  overflow: hidden;
  background-color: pink;
  /* 同步过渡效果 */
  transition: height 0.08s ease-out;
}

/* 拖拽手柄样式优化 */
.resize-handle {
  height: 6px;
  background-color: #e0e0e0;
  cursor: ns-resize;
  user-select: none;
  border-top: 1px solid #ccc;
  border-bottom: 1px solid #ccc;
  transition: all 0.2s; /* 合并过渡属性 */
  flex-shrink: 0; /* 防止被压缩 */
  position: relative;
  z-index: 10; /* 确保在最上层 */
}

/* 新增：添加视觉指引线 */
.resize-handle::before {
  content: "";
  position: absolute;
  left: 50%;
  top: 0;
  bottom: 0;
  width: 1px;
  background-color: rgba(0, 0, 0, 0.2);
}

.resize-handle:hover {
  background-color: #d0d0d0;
}

.resize-handle.dragging {
  background-color: #b0b0b0;
  height: 8px;
  /* 新增：拖拽时更明显的视觉反馈 */
  box-shadow: 0 0 0 1px rgba(0, 0, 0, 0.1);
}

/* 禁用元素拖拽 */
* {
  -webkit-user-drag: none;
  touch-action: none; /* 优化触摸设备体验 */
}

.resize-handle {
  pointer-events: auto;
}
</style>
