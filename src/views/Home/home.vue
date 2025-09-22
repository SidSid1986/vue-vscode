<template>
  <div class="container" ref="containerRef">
    <!-- 左侧：文件树，固定 25% 宽度 -->
    <div
      class="file-content"
      ref="fileContentRef"
      :style="{ width: leftPanelWidth }"
    >
      <FileTree @fileSelected="fileSelected" />
    </div>

    <!-- 左右拖拽手柄 -->
    <div
      class="drag-container-horizontal"
      @mousedown="startHorizontalDrag"
      :class="{ dragging: isHorizontalDragging }"
    ></div>

    <!-- 右侧：代码内容 + 拖拽 + Terminal，宽度 75% -->
    <div
      class="code-content"
      ref="codeContentRef"
      :style="{ width: rightPanelWidth }"
    >
      <!-- 上方：代码展示区，高度动态变化 -->
      <div class="editor-content" :style="{ height: editorHeight }">
        <MonacoCom
          ref="jsonComponents"
          :model-value="strJson"
          @update:model-value="handleChangeResponseJson"
        />
      </div>

      <!-- 中间：拖拽手柄 -->
      <div class="drag-container">
        <div
          class="resize-handle"
          @mousedown="startDrag"
          :class="{ dragging: isDragging }"
        ></div>

        <!-- 下方：Terminal 组件，高度动态变化 -->
        <div class="terminal-container" :style="{ height: terminalHeight }">
          <Terminal
            :terminalContentHeight="terminalContentHeight"
            @closeTerminalFunc="closeTerminalFunc"
          />
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, nextTick, computed } from "vue";
import FileTree from "@/components/FileTree.vue";
import MonacoCom from "@/components/MonacoCom.vue";
import Terminal from "@/components/Terminal.vue";

//  文件内容
const oneFileName = ref("");
const strJson = ref(
  "# Python 示例代码\nprint('Hello Python!')\nresult = 1 + 2\nprint('计算结果：', result)"
);

// 高度相关
const codeContentRef = ref(null);
const editorHeight = ref(""); // 动态高度，单位 px
const terminalHeight = ref(""); // 动态高度，单位 px
const terminalContentHeight = ref("");
const isDragging = ref(false);

// 拖拽相关变量（垂直拖拽）
const startY = ref(0);
const startTerminalHeight = ref(0); // 拖拽起始时 terminal 高度（px）

// 宽度相关
const containerRef = ref(null);
const fileContentRef = ref(null);
const leftPanelWidth = ref("15vw"); // 左侧面板宽度
const rightPanelWidth = ref("85vw"); // 右侧面板宽度
const isHorizontalDragging = ref(false);

// 水平拖拽相关变量
const startX = ref(0);
const startLeftWidth = ref(0); // 拖拽起始时左侧宽度（vw）

// 文件选择  ========
const fileSelected = (file) => {
  console.log("File selected:", file);
  oneFileName.value = file.name;
  strJson.value = file.content;
};

// 开始垂直拖拽（编辑器和终端之间的拖拽）
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

// 垂直拖拽中：计算新的 Terminal 和 Editor 高度
const handleMouseMove = (e) => {
  if (!isDragging.value) return;

  const container = codeContentRef.value;
  if (!container) return;

  const containerHeight = container.clientHeight;
  const deltaY = startY.value - e.clientY; //  startY - e.clientY

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

  terminalContentHeight.value = `${newTerminalHeightPx - 40}px`;
  editorHeight.value = `${newEditorHeightPx}px`;
};

// 停止垂直拖拽
const stopDrag = () => {
  isDragging.value = false;
  document.removeEventListener("mousemove", handleMouseMove);
  document.removeEventListener("mouseup", stopDrag);
};

// 开始水平拖拽（左右面板的拖拽）
const startHorizontalDrag = (e) => {
  if (!containerRef.value) return;

  isHorizontalDragging.value = true;

  const container = containerRef.value;
  const containerWidth = container.clientWidth;

  startX.value = e.clientX;
  startLeftWidth.value = parseWidth(leftPanelWidth.value, containerWidth);

  document.addEventListener("mousemove", onHorizontalMouseMove);
  document.addEventListener("mouseup", stopHorizontalDrag);

  e.preventDefault();
};

// 水平拖拽中：计算新的左右面板宽度
const onHorizontalMouseMove = (e) => {
  if (!isHorizontalDragging.value) return;

  if (!containerRef.value) return;

  const container = containerRef.value;
  const containerWidth = container.clientWidth;
  const mouseX = e.clientX;

  // 当前鼠标位置对应的左侧宽度百分比
  let newLeftWidthVw = (mouseX / containerWidth) * 100;

  // 限制范围，避免超出合理区间
  newLeftWidthVw = Math.max(10, Math.min(90, newLeftWidthVw)); // 限制在 10% ~ 90%

  const newRightWidthVw = 100 - newLeftWidthVw;

  //  基于像素做更严格的最小宽度限制
  /*
  const currentLeftPx = (newLeftWidthVw / 100) * containerWidth;
  const currentRightPx = (newRightWidthVw / 100) * containerWidth;
  if (currentLeftPx < MIN_LEFT_PX || currentRightPx < MIN_RIGHT_PX) {
    if (currentLeftPx < MIN_LEFT_PX) {
      newLeftWidthVw = (MIN_LEFT_PX / containerWidth) * 100;
    } else if (currentRightPx < MIN_RIGHT_PX) {
      newLeftWidthVw = 100 - (MIN_RIGHT_PX / containerWidth) * 100;
    }
    newRightWidthVw = 100 - newLeftWidthVw;
  }
  */

  leftPanelWidth.value = `${newLeftWidthVw}vw`;
  rightPanelWidth.value = `${newRightWidthVw}vw`;
};

// 停止水平拖拽
const stopHorizontalDrag = () => {
  isHorizontalDragging.value = false;
  document.removeEventListener("mousemove", onHorizontalMouseMove);
  document.removeEventListener("mouseup", stopHorizontalDrag);
};

// 转为 px 数字（高度）
const parseHeight = (heightStr, containerHeight) => {
  if (heightStr.endsWith("px")) {
    return parseFloat(heightStr);
  } else if (heightStr.endsWith("vh")) {
    return (parseFloat(heightStr) / 100) * containerHeight;
  }
  return containerHeight / 2; // 默认值，备用
};

// 转为数值（宽度，vw单位）
const parseWidth = (widthStr, containerWidth) => {
  if (widthStr.endsWith("vw")) {
    return parseFloat(widthStr);
  } else if (widthStr.endsWith("px")) {
    return (parseFloat(widthStr) / containerWidth) * 100;
  }
  return 25; // 默认25%
};

const closeTerminalFunc = () => {
  terminalHeight.value = 0;
  editorHeight.value = "100%";
};

const handleChangeResponseJson = () => {
  // 返回内容值，根据业务增加
};

onMounted(() => {
  nextTick(() => {
    // 初始化高度
    const containerHeight = codeContentRef.value?.clientHeight || 600;
    const initialTerminalHeight = containerHeight * 0.3; // 30%
    const initialEditorHeight = containerHeight - initialTerminalHeight;

    terminalHeight.value = `${initialTerminalHeight}px`;
    terminalContentHeight.value = `${initialTerminalHeight - 40}px`;
    editorHeight.value = `${initialEditorHeight}px`;
  });
});

onUnmounted(() => {
  document.removeEventListener("mousemove", handleMouseMove);
  document.removeEventListener("mouseup", stopDrag);
  document.removeEventListener("mousemove", handleHorizontalMouseMove);
  document.removeEventListener("mouseup", stopHorizontalDrag);
});
</script>

<style scoped lang="scss">
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
  height: 100vh;
  overflow-y: auto;
  overflow-x: hidden;
}

.code-content {
  height: 100vh;
  position: relative;
  overflow: hidden;
}

.editor-content {
  width: 100%;
  overflow: auto;
  box-sizing: border-box;
  background: #f6f6f6;
}

.drag-container-horizontal {
  width: 8px;
  background-color: #1e1e1e;
  cursor: ew-resize;
  user-select: none;
  position: relative;
  z-index: 20;
  transition: background-color 0.2s;
  flex-shrink: 0;
  height: 100%;
  align-self: stretch;
}

.drag-container-horizontal:hover {
  background-color: #007fd4;
}

.drag-container-horizontal.dragging {
  background-color: #007fd4;
}

.drag-container {
  width: 100%;
  position: absolute;
  bottom: 0;
}

.terminal-container {
  width: 100%;
  box-sizing: border-box;
  overflow: hidden;
  background-color: #1e1e1e;
}

.resize-handle {
  height: 8px;
  background-color: transparent;
  cursor: ns-resize;
  user-select: none;
  transition: background-color 0.2s;
  flex-shrink: 0;
  position: relative;
  z-index: 10;
  border: 1px solid transparent;
  box-sizing: border-box;
}

.resize-handle:hover {
  background-color: #007fd4;
  border: 1px solid transparent;
}

.resize-handle.dragging {
  background-color: #007fd4;
  border: 1px solid transparent;
  height: 8px;
}

/* 防止文本选择与拖拽冲突 */
* {
  -webkit-user-drag: none;
  touch-action: none;
}
</style>
