<template>
  <!-- 最外层布局容器，100vw / 100vh，flex column -->
  <div class="app-wrapper">
    <!--  顶部工具栏：黄色，固定 50px -->
    <div class="global-toolbar">
      <!-- 顶部工具栏{{ leftPanelWidth }}--{{ rightPanelWidth }} -->
      <!-- <ToolMenu :menuData="menuData" /> -->
      <ToolMenu />
    </div>

    <!--  container（占满剩余所有高度） -->
    <div class="container" ref="containerRef">
      <!-- 左侧粉色区域（文件操作图标区） -->
      <div class="left-icon-area">
        <i class="iconfont icon-icon-test1" @click="openFileContent"></i>
        <i class="iconfont icon-icon-test"></i>
      </div>

      <!-- 左侧文件树 -->
      <div
        class="file-content"
        ref="fileContentRef"
        :style="{ width: leftPanelWidth }"
      >
        <FileTree @fileSelected="fileSelected" />
      </div>

      <!--  左右拖拽手柄 -->
      <div
        class="drag-container-horizontal"
        @mousedown="startHorizontalDrag"
        :class="{ dragging: isHorizontalDragging }"
      ></div>

      <!--  右侧代码 + Terminal 区域 -->
      <div
        class="code-content"
        ref="codeContentRef"
        :style="{ width: rightPanelWidth }"
      >
        <div class="code-tab">
          <div v-for="item in selectedFileArr" :key="item.id">
            <div
              @click="tabClick(item)"
              class="tab-item"
              :class="
                item.id == selectedId ? 'tab-selected' : 'tab-no-selected'
              "
            >
              <span
                :class="item.selected ? '' : 'text-selected'"
                class="tab-text"
                >{{ item.fileName }}</span
              >
              <i
                class="iconfont icon-guanbi"
                :class="
                  item.id == selectedId ? 'icon-selected' : 'icon-no-selected'
                "
              ></i>
            </div>
          </div>
        </div>
        <!-- 上方：代码展示区，高度动态变化 -->
        <div class="editor-content" :style="{ height: editorHeight }">
          <MonacoCom
            ref="jsonComponents"
            :model-value="selectedJson"
            @update:model-value="handleChangeResponseJson"
            :language="selectedLanguage"
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

    <!-- ✅ 3. 新增：底部工具栏：蓝色，固定 30px -->
    <div class="global-status-bar">🧩 底部工具栏</div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, nextTick, computed } from "vue";
import FileTree from "@/components/FileTree.vue";
import MonacoCom from "@/components/MonacoCom.vue";
import Terminal from "@/components/Terminal.vue";
import ToolMenu from "@/components/ToolMenu.vue";

const menuData = [
  {
    id: 1,
    name: "文件",
    show: false,
    children: [
      {
        id: 11,
        name: "打开文件夹",
        children: [],
      },
      {
        id: 12,
        name: "打开文件",
        children: [],
      },
    ],
  },
  {
    id: 2,
    name: "编辑",
    show: false,
    children: [
      {
        id: 22,
        name: "查找",
        children: [],
      },
    ],
  },
];

//  （leftPanel + rightPanel）总宽度 = 100 - 3 - 0.4 = 96.6vw
const DYNAMIC_TOTAL_WIDTH_VW = 96.6;
const previousLeftWidthVw = ref(15); // 默认值 15，万一从未记录过，也合理

//  文件内容
// const fileName = ref("");
// const strJson = ref(
//   "# Python 示例代码\nprint('Hello Python!')\nresult = 1 + 2\nprint('计算结果：', result)"
// );
const selectedId = ref("");
// const selectedFile = ref({
//   fileName: "",
//   strJson: "",
//   selected: false,
//   id: "",
//   language: "",
// });
const selectedLanguage = ref("");
const selectedJson = ref("");
const selectedFileArr = ref([]);

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
const rightPanelWidth = ref("81.6vw"); // 右侧面板宽度
const isHorizontalDragging = ref(false);

// 水平拖拽相关变量
const startX = ref(0); //鼠标按下时的 X 坐标
const startLeftWidthVw = ref(0); //拖拽开始时，左侧面板的宽度

// 文件选择  ========
const fileSelected = (file) => {
  console.log("File selected:", file);

  selectedId.value = file.id;
  selectedJson.value = file.content;
  selectedLanguage.value = getFileLanguage(file.name);

  // 1. 获取语言
  const language = getFileLanguage(file.name);

  // 2. 构造新文件对象（但注意：我们可能不新增，而是更新已有的）
  const newFile = {
    fileName: file.name,
    strJson: file.content,
    id: file.id,
    selected: file.selected,
    language: language,
  };

  // 3. 查找是否已存在相同 id 的文件
  const existingIndex = selectedFileArr.value.findIndex(
    (item) => item.id === newFile.id
  );

  if (existingIndex !== -1) {
    const existingFile = selectedFileArr.value[existingIndex];

    console.log(
      `🔍 已存在相同 id 的文件：${newFile.fileName}，当前选中状态：${existingFile.selected}，新传入状态：${newFile.selected}`
    );

    if (existingFile.selected === false && newFile.selected === true) {
      // ✅ 情况：已存在项是预览态（false），但用户双击了它（true）→ 升级为正式选中
      console.log(`🔄 将文件从预览态升级为正式选中：${newFile.fileName}`);

      // 直接更新该索引的 selected 状态为 true，其他信息也可以同步更新（比如 content / language）
      selectedFileArr.value[existingIndex] = {
        ...existingFile,
        strJson: newFile.strJson, // 确保内容最新
        selected: true, // 升级为正式选中
        language: language, // 确保语言正确
      };

      console.log("✅ 文件状态已升级为 selected: true（正式打开）");
    } else {
      // 其它情况，比如：
      // - 已存在且 selected: true，又传入了 selected: true（双击同一个文件）
      // - 或者已存在 selected: false，又传入了 selected: false（重复单击）
      // 你可以选择更新内容，或者什么都不做

      console.log(
        `ℹ️ 文件已存在，且状态未发生变化或不符合升级条件，可选择更新内容。当前状态：${existingFile.selected}`
      );

      // 【可选】如果你希望无论如何都更新内容，可以取消下面注释：
      // selectedFileArr.value[existingIndex] = newFile;
    }

    return; // 已处理 id 相同的情况，无需新增
  }

  // ===========================
  // 4. 如果 id 不存在，则执行原来的 “添加逻辑”
  // ===========================

  // 4.1 【仅针对 selected: false 的文件】保证最多只有一个预览态
  if (newFile.selected === false) {
    const hasInactiveFile = selectedFileArr.value.some(
      (item) => item.selected === false
    );

    if (hasInactiveFile) {
      const inactiveIndex = selectedFileArr.value.findIndex(
        (item) => item.selected === false
      );
      if (inactiveIndex !== -1) {
        selectedFileArr.value.splice(inactiveIndex, 1); // 移除旧的未选中文件
      }
    }

    selectedFileArr.value.push(newFile);
    console.log(
      "添加了一个 seleced: false 的文件（预览态）：",
      newFile.fileName
    );
  } else {
    // 4.2 selected: true（双击 / 正式选中），直接添加，无限制
    selectedFileArr.value.push(newFile);
    console.log(
      "添加了一个 seleced: true 的文件（正式选中）：",
      newFile.fileName
    );
  }
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
  document.body.style.cursor = "ns-resize";
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
  const deltaY = startY.value - e.clientY;

  let newTerminalHeightPx = startTerminalHeight.value + deltaY;

  const totalHeight = containerHeight;

  const minTerminalHeight = 100;
  const minEditorHeight = 100;

  newTerminalHeightPx = Math.max(
    minTerminalHeight,
    Math.min(newTerminalHeightPx, totalHeight - minEditorHeight)
  );

  const newEditorHeightPx = totalHeight - newTerminalHeightPx;

  // ✅ 关键：减去 code-tab 的高度 4vh（和 onMounted 保持一致！）
  const codeTabHeightVh = 4;
  const codeTabHeightPx = (codeTabHeightVh / 100) * containerHeight;

  const finalEditorHeightPx = newEditorHeightPx - codeTabHeightPx;

  terminalHeight.value = `${newTerminalHeightPx}px`;
  terminalContentHeight.value = `${newTerminalHeightPx - 40}px`;
  editorHeight.value = `${finalEditorHeightPx - 1}px`; //  减去 1px 视觉补偿
};
// 停止垂直拖拽
const stopDrag = () => {
  isDragging.value = false;
  document.body.style.cursor = "";

  document.removeEventListener("mousemove", handleMouseMove);
  document.removeEventListener("mouseup", stopDrag);
};

// 开始水平拖拽（左右面板的拖拽）
const startHorizontalDrag = (e) => {
  // 如果 containerRef（即 .container）还未挂载或不存在，直接退出函数，避免报错
  if (!containerRef.value) return;

  // 标记当前正在水平拖拽中，用于控制状态和样式
  isHorizontalDragging.value = true;

  // 获取容器 DOM 元素（即 .container，也就是整体 100vw 的父容器）
  const container = containerRef.value;

  // 获取容器的 clientWidth（当前视口下容器的实际像素宽度，用于后续计算比例）
  const containerWidth = container.clientWidth;

  // 记录鼠标按下的 X 坐标，用于后续计算鼠标移动的距离（deltaX）
  startX.value = e.clientX;

  // ✅ 解析当前 leftPanelWidth 的值（比如 "15vw"）→ 转为数字 15
  // 这个函数会根据 "15vw" 这种格式，提取出数字部分 15，用作拖拽的起始宽度
  startLeftWidthVw.value = parseWidth(leftPanelWidth.value, containerWidth);

  // 设置鼠标样式为 ew-resize（左右箭头），提示用户当前可以左右拖拽
  document.body.style.cursor = "ew-resize";

  // 添加鼠标移动事件监听：拖拽过程中会持续触发 onHorizontalMouseMove
  document.addEventListener("mousemove", onHorizontalMouseMove);

  // 添加鼠标释放事件监听：鼠标松开时触发 stopHorizontalDrag，结束拖拽
  document.addEventListener("mouseup", stopHorizontalDrag);

  // 阻止默认行为，比如避免拖拽时选中文本或触发其他浏览器默认拖拽行为
  e.preventDefault();
};
// 水平拖拽中：计算新的左右面板宽度
const onHorizontalMouseMove = (e) => {
  if (!isHorizontalDragging.value) return;

  const deltaX = e.clientX - startX.value;
  const container = containerRef.value;
  const containerWidth = container.clientWidth;

  // 1. 计算原始的拖拽后宽度（可能 <10，可能很大）
  let rawNewLeftWidthVw =
    startLeftWidthVw.value + (deltaX / containerWidth) * 100;

  // 2. 限制右侧最小宽度为 15vw → 即左侧最大为 81.6vw
  const MIN_RIGHT_WIDTH_VW = 15;
  const MAX_LEFT_WIDTH_VW = DYNAMIC_TOTAL_WIDTH_VW - MIN_RIGHT_WIDTH_VW; // 96.6 - 15 = 81.6

  // 先限制左侧最大宽度，避免右侧小于 15
  rawNewLeftWidthVw = Math.min(MAX_LEFT_WIDTH_VW, rawNewLeftWidthVw);

  // 3. 定义左侧最小可见宽度为 10vw
  const MIN_LEFT_VISIBLE_VW = 10;

  let newLeftWidthVw;

  // 4. 核心交互逻辑：卡住 or 吸附隐藏
  if (rawNewLeftWidthVw < MIN_LEFT_VISIBLE_VW - 7) {
    //  继续往左拖拽，导致宽度 < 3 → 触发吸附隐藏，用力拖拽
    newLeftWidthVw = 0; // 左侧隐藏
  } else {
    //  限制最小为 10，实现 “卡住” 效果（拖到 10 就停住）
    newLeftWidthVw = Math.max(MIN_LEFT_VISIBLE_VW, rawNewLeftWidthVw);
  }

  // 5. 更新左右面板宽度
  leftPanelWidth.value = `${newLeftWidthVw.toFixed(4)}vw`;
  rightPanelWidth.value = `${(DYNAMIC_TOTAL_WIDTH_VW - newLeftWidthVw).toFixed(
    4
  )}vw`;
};
// 停止水平拖拽
const stopHorizontalDrag = () => {
  // 标记拖拽状态结束，用于 UI 状态控制和后续逻辑判断
  isHorizontalDragging.value = false;

  // 恢复鼠标指针为默认样式，不再显示 ew-resize（左右箭头）
  document.body.style.cursor = "";

  // 移除鼠标移动事件监听，避免拖拽结束后仍持续响应 mousemove
  document.removeEventListener("mousemove", onHorizontalMouseMove);

  // 移除鼠标释放事件监听（其实这一步可以省略，因为 mouseup 只触发一次，但为了规范，保留它）
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

const openFileContent = () => {
  console.log("当前 leftPanelWidth.value:", leftPanelWidth.value);

  // ✅ 新增：专门处理拖拽导致的 0.0000vw 情况
  if (leftPanelWidth.value === "0.0000vw") {
    //  1：用户拖拽直接到 0.0000vw（不是点击按钮隐藏的）
    // 直接恢复成默认的 15vw，不记录、不隐藏、不走其他逻辑
    console.log("检测到拖拽导致的 0.0000vw，直接恢复为默认 15vw");

    leftPanelWidth.value = "15.00vw";
    rightPanelWidth.value = "81.60vw";
  } else if (leftPanelWidth.value !== "0vw") {
    //   2：左侧是显示的（比如 "15vw"、"18.5vw"、"20vw" 等正常值）
    // 正常逻辑：记录当前宽度，然后隐藏成 "0vw"

    console.log("当前是显示状态，记录宽度然后隐藏");

    const currentWidthStr = leftPanelWidth.value;
    let currentWidthVw = 0;

    if (currentWidthStr.endsWith("vw")) {
      currentWidthVw = parseFloat(currentWidthStr);
    } else {
      currentWidthVw = 15; // 安全默认值
    }

    // 记录当前宽度，供之后恢复
    previousLeftWidthVw.value = currentWidthVw;

    // 隐藏左侧面板
    leftPanelWidth.value = "0vw";
    rightPanelWidth.value = "96.6vw";
  } else {
    // 🔸 情况 3：左侧是隐藏的（leftPanelWidth === "0vw"）
    // 正常逻辑：恢复之前记录的宽度

    console.log("恢复之前记录的宽度:", previousLeftWidthVw.value);

    const prevWidthVw = previousLeftWidthVw.value || 15; // 如果没记录过，用 15
    leftPanelWidth.value = `${prevWidthVw.toFixed(2)}vw`; // 例如 "18.50vw"
    const rightWidthVw = DYNAMIC_TOTAL_WIDTH_VW - prevWidthVw;
    rightPanelWidth.value = `${rightWidthVw.toFixed(2)}vw`; // 例如 "78.10vw"
  }
};
const handleChangeResponseJson = () => {
  // 返回内容值，根据业务增加
};

const tabClick = (item) => {
  console.log(item);
  selectedId.value = item.id;
  selectedJson.value = item.strJson;
  selectedLanguage.value = getFileLanguage(item.fileName);
};

// 🧠 工具函数：根据文件名返回 Monaco Editor 对应的语言 mode
function getFileLanguage(fileName) {
  console.log(fileName);
  const ext = fileName.split(".").pop()?.toLowerCase(); // 获取文件后缀，如 'py', 'css', 'js'

  const languageMap = {
    // ✅ 常见文件后缀与 Monaco Editor 的 language mode 对照
    js: "javascript",
    ts: "typescript",
    json: "json",
    html: "html",
    css: "css",
    scss: "scss",
    less: "less",
    py: "python",
    java: "java",
    cpp: "cpp",
    c: "c",
    go: "go",
    rust: "rust",
    php: "php",
    sql: "sql",
    md: "markdown",
    xml: "xml",
    yaml: "yaml",
    yml: "yaml",
    sh: "shell",
    bash: "shell",
    // 可继续扩展...
  };

  return languageMap[ext] || "plaintext"; // 如果没匹配到，默认使用 plaintext（纯文本）
}

onMounted(() => {
  nextTick(() => {
    const containerHeight = codeContentRef.value?.clientHeight || 600;
    const codeTabHeightVh = 4;
    const codeTabHeightPx = (codeTabHeightVh / 100) * containerHeight;

    const initialTerminalHeight = containerHeight * 0.3; // 30%
    const initialEditorHeight =
      containerHeight - initialTerminalHeight - codeTabHeightPx; // ✅ 减去 code-tab 高度

    terminalHeight.value = `${initialTerminalHeight}px`;
    terminalContentHeight.value = `${initialTerminalHeight - 40}px`;
    editorHeight.value = `${initialEditorHeight - 1}px`; // ✅ 现在 editorHeight 不包含 4vh 了
  });
});

onUnmounted(() => {
  document.removeEventListener("mousemove", handleMouseMove);
  document.removeEventListener("mouseup", stopDrag);
  // document.removeEventListener("mousemove", handleHorizontalMouseMove);
  document.removeEventListener("mouseup", stopHorizontalDrag);
});
</script>

<style scoped lang="scss">
/* 最外层：100vw / 100vh，column 布局 */
.app-wrapper {
  width: 100vw;
  height: 100vh;
  display: flex;
  flex-direction: column;
  margin: 0;
  padding: 0;
  box-sizing: border-box;
  font-family: sans-serif;
  overflow: hidden;
}

/* 1. 顶部工具栏：黄色，固定 50px */
.global-toolbar {
  height: 50px;
  // background-color: blue;
  background-color: #222223;
  color: #ffffff;
  display: flex;
  align-items: center;
  padding: 0 16px;
  font-weight: bold;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  z-index: 1000;
}

/* 2. 你原来的 .container，占满剩余高度 */
.container {
  display: flex;
  flex-direction: row;
  width: 100%;
  height: 100%; /* 占满除去顶部和底部之后的所有高度 */
  overflow: hidden;
  user-select: none;
  -webkit-user-select: none;
}

/* 3. 你新增的：左侧粉色区域（文件操作图标区） */
.left-icon-area {
  width: 3vw;
  background-color: #222223;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: flex-start;
  padding: 10px 0;

  i {
    color: #707071;
    font-size: 30px;
    margin-bottom: 30px;
    cursor: pointer;
    font-weight: bold;
  }
  i:hover {
    color: #e6e6e6;
  }
}

.file-content {
  height: 100%;
  overflow-y: auto;
  overflow-x: hidden;
}

.code-content {
  height: 100%;
  position: relative;
  overflow: hidden;
  box-sizing: border-box;
  // border: 2px solid red;
}

.code-tab {
  width: 100%;
  height: 4vh;
  background-color: #242526;
  box-sizing: border-box;
  display: flex;
  flex-direction: row;
  justify-content: flex-start;
  align-items: center;
  .tab-item {
    height: 4vh;
    line-height: 4vh;

    color: #ffffff;
    cursor: pointer;
    padding: 0 10px;
    font-size: 16px;
    display: flex;
    flex-direction: row;
    justify-content: space-between;
    align-items: center;
    box-sizing: border-box;
  }
  .text-selected {
    // color: red;
    font-style: italic;
  }
  .tab-text {
    // border: 1px solid red;
    height: 3vh;
    line-height: 3vh;
    margin: 0px 5px;
  }
  .tab-selected {
    background-color: #292a2b;
    border-bottom: 1px solid #19f9d8;
    color: #19f9d8;
  }

  .tab-no-selected {
    background-color: #222223;
  }

  .icon-selected {
    color: #19f9d8;
    font-size: 12px;
    border: 1px solid #292a2b;
    height: 2vh;
    width: 2vh;
    text-align: center;
    line-height: 2vh;
  }
  .icon-selected:hover {
    border: 1px solid #ffffff;
  }

  .icon-no-selected {
    color: #222223;
    font-size: 12px;
    border: 1px solid #222223;
    height: 2vh;
    width: 2vh;
    text-align: center;
    line-height: 2vh;
  }

  .icon-no-selected:hover {
    color: #ffffff;
    border: 1px solid #ffffff;
  }
}

.editor-content {
  width: 100%;
  height: 100%;
  overflow: auto;
  box-sizing: border-box;
  background-color: #292a2b;
  // border: 2px solid green;
}

.drag-container-horizontal {
  width: 0.4vw;
  background-color: #242526;
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
  width: 0.4vw;
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
  background-color: #292a2b;
  // border:1px solid red;
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

* {
  -webkit-user-drag: none;
  touch-action: none;
}

/* ✅ 4. 新增：底部工具栏：蓝色，固定 30px */
.global-status-bar {
  height: 30px;
  background-color: blue; /* 你要求的蓝色 */
  color: white;
  display: flex;
  align-items: center;
  padding: 0 16px;
  font-size: 12px;
  z-index: 1000;
}
</style>

<style lang="scss">
.file-content {
  /* 滚动条整体 */
  ::-webkit-scrollbar {
    width: 20px;
    height: 20px;
  }

  /* 滚动条轨道 */
  ::-webkit-scrollbar-track {
    background: #292a2b;
    border-radius: 0;
  }

  /* 滚动条滑块 */
  ::-webkit-scrollbar-thumb {
    background-color: #313438;
    border-radius: 0;
    border: 2px solid transparent;
    background-clip: content-box;
    transition: background-color 0.2s ease; /* 添加过渡效果 */
  }

  /* 滚动条滑块悬停状态 */
  ::-webkit-scrollbar-thumb:hover {
    background-color: #757575 !important;
    cursor: pointer;
  }

  /* 滚动条滑块激活状态（点击时） */
  ::-webkit-scrollbar-thumb:active {
    background-color: #757575 !important; /* 使用稍暗的红色区分激活状态 */
  }

  /* 滚动条角落 */
  ::-webkit-scrollbar-corner {
    background: #292a2b; /* 与轨道颜色保持一致 */
  }
}
</style>
