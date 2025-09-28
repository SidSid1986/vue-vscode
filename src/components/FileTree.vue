<template>
  <div class="file-tree">
    <!-- #45A9F9 -->

    <div v-if="items.length === 0" class="open-btn">
      <el-button type="primary" @click="selectFolder">
        测试选择文件夹
      </el-button>
    </div>

    <div v-if="items.length > 0" class="tree-container">
      <!-- @onSelect="onItemSelected" -->
      <vue3-tree-vue
        :items="items"
        :isCheckable="false"
        :hideGuideLines="false"
        @onCheck="onItemChecked"
        @onExpand="onItemExpanded"
        @dropValidator="onBeforeItemDropped"
        class="norem-tree"
      >
        <!-- 用 FileIcon 组件渲染图标 -->
        <template v-slot:item-prepend-icon="treeItem">
          <div
            class="file-icon-border"
            @click="handleClick(treeItem)"
            @dblclick="handleDoubleClick(treeItem)"
          >
            <FileIcon
              :fileName="treeItem.name || 'default_file'"
              :fileType="treeItem.type || 'file'"
              :isOpen="treeItem.expanded || false"
              :size="20"
            />
          </div>
        </template>
        <template v-slot:item-name="treeItem">
          <span
            @click="handleClick(treeItem)"
            @dblclick="handleDoubleClick(treeItem)"
            class="tree-node-name"
            :class="selectedId === treeItem.id ? 'selected-css' : ''"
          >
            {{ treeItem.name }}
          </span>
        </template>
      </vue3-tree-vue>
    </div>
  </div>
</template>

<script setup>
import { nextTick, ref } from "vue";

import FileIcon from "@/components/FileIcon.vue";

import Vue3TreeVue from "vue3-tree-vue";
import "vue3-tree-vue/dist/style.css";
import { v4 as uuidv4 } from "uuid";

const items = ref([]);
const emits = defineEmits(["fileSelected"]);
const selectedFileName = ref("");
const selectedFileContent = ref("");
// 防抖变量
const clickTimer = ref(null);
const isDoubleClick = ref(false);
const allFiles = ref([]);

const selectedId = ref(null);

// 选择文件夹
const selectFolder = () => {
  const input = document.createElement("input");
  input.type = "file";
  input.webkitdirectory = true;
  input.onchange = (e) => {
    const target = e.target;
    if (target && target.files) {
      const files = Array.from(target.files);
      console.log(`总共选择了 ${files.length} 个文件`);

      // 保存所有文件，用于后续懒加载
      allFiles.value = files;

      console.log(files);

      // 只构建 vue-vscode 下的第一层结构（懒加载优化）
      const tree = buildTreeStructure(files);

      console.log(tree);
      items.value = tree;
    }
  };
  input.click();
};

//选择文件

// 构建树结构：动态识别第一层目录，只构建该目录下的第一层子节点（如 public、src、views、main.js）
const buildTreeStructure = (files) => {
  console.log(files);
  if (!files || files.length === 0) {
    console.log("没有文件，返回空树");
    return [];
  }

  // 1. 获取所有文件的 webkitRelativePath
  const paths = Array.from(files).map((file) => file.webkitRelativePath);

  // 2. 找出所有“第一层目录”（如 vue-vscode、src、my-app，即第一个 / 之前的部分）
  const firstLevelDirsSet = new Set();
  paths.forEach((path) => {
    const firstSlashIndex = path.indexOf("/");
    if (firstSlashIndex === -1) return; // 没有 / ，说明没有第一层目录
    const firstLevelDir = path.slice(0, firstSlashIndex); // 如 vue-vscode
    firstLevelDirsSet.add(firstLevelDir);
  });

  const firstLevelDirs = Array.from(firstLevelDirsSet);
  console.log("检测到的第一层目录：", firstLevelDirs);

  // 3. 如果没有第一层目录（所有文件在根目录下），构建扁平结构
  if (firstLevelDirs.length === 0) {
    console.log("没有第一层目录，所有文件在根下，构建扁平结构");
    return buildFlatStructure(paths);
  }

  // 4. 只处理第一个第一层目录（如 vue-vscode），如需支持多个可遍历 firstLevelDirs
  const rootFirstLevelDir = firstLevelDirs[0]; // 如 vue-vscode
  const rootPrefix = rootFirstLevelDir + "/"; // 如 vue-vscode/

  // 5. 筛选该第一层目录下的所有文件/文件夹路径（如 vue-vscode/public、vue-vscode/src、vue-vscode/main.js）
  const rootFiles = paths.filter((path) => path.startsWith(rootPrefix));
  console.log("筛选后的第一层目录下的文件/文件夹：", rootFiles);

  const rootNodeMap = new Map(); // 用于去重

  rootFiles.forEach((path) => {
    // 6. 去掉第一层前缀，得到相对路径（如 public/icons/file_type_angular.svg 或 main.js）
    const relativePath = path.slice(rootPrefix.length);

    // 7. 分割成路径部分，如 ["public", "icons", "file_type_angular.svg"] 或 ["main.js"]
    const pathParts = relativePath.split("/");

    // 8. 取第一级子项名称，如 "public"、"src"、"main.js" → pathParts[0]
    const nodeName = pathParts[0]; // ✅ 这才是你要构建树节点的名称

    // 9. 判断是文件夹还是文件：有 . 就是文件，没有就是文件夹
    const fileName = nodeName; // 当前节点的名称，如 public、main.js
    const hasExtension = fileName.includes("."); // 是否包含扩展名
    const isDirectory = !hasExtension; // 没有 . 才是文件夹 ✅
    const type = isDirectory ? "folder" : "file"; // ✅
    const ext = isDirectory
      ? ""
      : fileName.split(".").pop()?.toLowerCase() || ""; // ✅ 从文件名提取扩展名

    // 10. 构建节点的唯一路径（用于展示，如 vue-vscode/public）
    const fullPath = rootPrefix + nodeName;

    // 11. 创建节点对象（如果是文件，尝试绑定具体 file 对象）
    if (!rootNodeMap.has(nodeName)) {
      const node = {
        name: nodeName,
        path: nodeName,
        isDirectory: isDirectory,
        file: isDirectory
          ? null
          : files.find((f) => f.webkitRelativePath === path), // 只有文件才绑定
        type: type,
        ext: ext,
        children: [],
        expanded: false,
        fullPath: fullPath,
      };
      rootNodeMap.set(nodeName, node);
    }
  });

  // 12. 转为数组并返回
  const treeNodes = Array.from(rootNodeMap.values());

  //排序
  treeNodes.sort((a, b) => {
    // 1. folder 排在 file 前面
    if (a.type === "folder" && b.type === "file") {
      return -1; // a（folder）排在前面
    }
    if (a.type === "file" && b.type === "folder") {
      return 1; // b（folder）排在前面
    }

    // 2. 如果都是 folder 或都是 file，按 name 字母升序排序（不区分大小写）
    const nameA = a.name.toLowerCase();
    const nameB = b.name.toLowerCase();
    if (nameA < nameB) {
      return -1;
    }
    if (nameA > nameB) {
      return 1;
    }

    // 名称相同则保持原序
    return 0;
  });

  const processedTreeNodes = treeNodes.map((node) => {
    if (
      node.type === "folder" &&
      Array.isArray(node.children) &&
      node.children.length === 0
    ) {
      return {
        ...node,
        children: [{ name: "(空)", type: "placeholder" }],
      };
    }
    return node;
  });

  // const treeNodes = [
  //   {
  //     name: "api",
  //     path: "api",
  //     isDirectory: true,
  //     file: null,
  //     type: "folder",
  //     ext: "",
  //     children: [{ name: "(空)", type: "placeholder" } ],
  //     expanded: false,
  //     fullPath: "src/api",
  //     id: "728e6eb6-e969-4be9-981e-0b1e8fa36eec",
  //   },
  //   {
  //     name: "assets",
  //     path: "assets",
  //     isDirectory: true,
  //     file: null,
  //     type: "folder",
  //     ext: "",
  //     children: [],
  //     expanded: false,
  //     fullPath: "src/assets",
  //     id: "9b636bf9-cecf-496d-91e4-45d2aa3ff9e9",
  //   },
  //   {
  //     name: "components",
  //     path: "components",
  //     isDirectory: true,
  //     file: null,
  //     type: "folder",
  //     ext: "",
  //     children: [],
  //     expanded: false,
  //     fullPath: "src/components",
  //     id: "6f0a78f4-b34d-4a7c-ab90-1e3f7e35ff2f",
  //   },
  //   {
  //     name: "router",
  //     path: "router",
  //     isDirectory: true,
  //     file: null,
  //     type: "folder",
  //     ext: "",
  //     children: [],
  //     expanded: false,
  //     fullPath: "src/router",
  //     id: "ac342a07-74f8-417f-952c-cd010cdea816",
  //   },
  //   {
  //     name: "store",
  //     path: "store",
  //     isDirectory: true,
  //     file: null,
  //     type: "folder",
  //     ext: "",
  //     children: [],
  //     expanded: false,
  //     fullPath: "src/store",
  //     id: "29492285-2647-4b85-9779-0822d482cfbd",
  //   },
  //   {
  //     name: "styles",
  //     path: "styles",
  //     isDirectory: true,
  //     file: null,
  //     type: "folder",
  //     ext: "",
  //     children: [],
  //     expanded: false,
  //     fullPath: "src/styles",
  //     id: "aa529bd9-ab26-4a2f-8fd0-c6515bc3fcee",
  //   },
  //   {
  //     name: "utils",
  //     path: "utils",
  //     isDirectory: true,
  //     file: null,
  //     type: "folder",
  //     ext: "",
  //     children: [],
  //     expanded: false,
  //     fullPath: "src/utils",
  //     id: "492f3f07-a9ec-466e-951f-2a9a903bae40",
  //   },
  //   {
  //     name: "views",
  //     path: "views",
  //     isDirectory: true,
  //     file: null,
  //     type: "folder",
  //     ext: "",
  //     children: [],
  //     expanded: false,
  //     fullPath: "src/views",
  //     id: "c936e2b8-cd07-4d84-9da9-30cb2058c25b",
  //   },
  //   {
  //     name: "App.vue",
  //     path: "App.vue",
  //     isDirectory: false,
  //     file: {},
  //     type: "file",
  //     ext: "vue",
  //     children: [],
  //     expanded: false,
  //     fullPath: "src/App.vue",
  //     id: "1bb23932-cc22-4301-b839-6a2f89577846",
  //   },
  //   {
  //     name: "main.js",
  //     path: "main.js",
  //     isDirectory: false,
  //     file: {},
  //     type: "file",
  //     ext: "js",
  //     children: [],
  //     expanded: false,
  //     fullPath: "src/main.js",
  //     id: "515dd652-4023-46d7-bc27-75b1b51ee100",
  //   },
  //   {
  //     name: "test.js",
  //     path: "test.js",
  //     isDirectory: false,
  //     file: {},
  //     type: "file",
  //     ext: "js",
  //     children: [],
  //     expanded: false,
  //     fullPath: "src/test.js",
  //     id: "260141f0-71fd-4440-a2da-01fa99fa31c2",
  //   },
  // ];
  console.log("构建的树节点（第一层子项）：", treeNodes);
  console.log("构建的树节点数：", treeNodes.length);
  console.log("加了占位的", processedTreeNodes);

  return processedTreeNodes;
};

// 扁平结构备用：当没有第一层目录时（所有文件在根下）
const buildFlatStructure = (paths) => {
  const nodeMap = new Map();

  paths.forEach((path) => {
    const fileName = path.split("/").pop(); // 如 App.vue
    if (!fileName) return;

    if (!nodeMap.has(fileName)) {
      const hasExtension = fileName.includes(".");
      const node = {
        name: fileName,
        path: fileName,
        isDirectory: !hasExtension,
        file: null,
        type: hasExtension ? "file" : "folder",
        ext: hasExtension ? fileName.split(".").pop()?.toLowerCase() || "" : "",
        children: [],
        expanded: false,
        fullPath: "",
        id: uuidv4(), // ✅ 每个节点唯一 ID，标准方式,
      };
      nodeMap.set(fileName, node);
    }
  });

  return Array.from(nodeMap.values());
};

// 处理点击（单击）
const handleClick = (treeItem) => {
  console.log(treeItem);
  if (isDoubleClick.value) {
    // 如果是双击，则忽略单击
    isDoubleClick.value = false; //  重置 isDoubleClick
    clearTimeout(clickTimer.value); //  清除残留的定时器
    return;
  }

  // 300ms 后判断是否是双击
  clickTimer.value = setTimeout(() => {
    if (!isDoubleClick.value) {
      // 300ms 内没有第二次点击，执行单击逻辑
      onSingleClick(treeItem);
    }
    clickTimer.value = null; //  清除定时器引用
    isDoubleClick.value = false; //  重置 isDoubleClick
  }, 200);
};

// 处理双击
const handleDoubleClick = (treeItem) => {
  clearTimeout(clickTimer.value); // 清除单击的定时器
  isDoubleClick.value = true; // 标记为双击
  onDoubleClick(treeItem); // 执行双击逻辑

  //  双击后，确保下一次单击能正常执行
  setTimeout(() => {
    isDoubleClick.value = false; // 300ms 后重置 isDoubleClick
  }, 300);
};

// 单击
const onSingleClick = (item) => {
  console.log("单击:", item);

  if (item.type === "file" && item.file) {
    selectedId.value = item.id;

    // 单击文件：读取文件内容
    selectedFileName.value = item.name;
    const file = item.file;

    const reader = new FileReader();
    reader.readAsText(file);

    reader.onload = (event) => {
      selectedFileContent.value = event.target.result;
      emits("fileSelected", {
        name: selectedFileName.value,
        content: selectedFileContent.value,
        id: item.id,
        selected: false,
      });
    };

    reader.onerror = () => {
      selectedFileContent.value = `无法读取文件：${reader.error.message}`;
    };
  } else if (item.type === "folder") {
    console.log("单击文件夹:", item);

    if (!item.expanded) {
      console.log(`📦 懒加载文件夹 "${item.name}" 的子内容`);

      // 1. 懒加载子节点
      const lazyChildren = loadLazyChildrenForFolder(item);
      console.log(lazyChildren);

      const processedTreeNodes = lazyChildren.map((node) => {
        if (
          node.type === "folder" &&
          Array.isArray(node.children) &&
          node.children.length === 0
        ) {
          return {
            ...node,
            children: [{ name: "(空)", type: "placeholder" }],
          };
        }
        return node;
      });

      //增加占位

      // 2.  不要直接修改 item.children，
      //    而是创建一个新的对象副本，包含更新后的 children
      const updatedItem = { ...item, children: processedTreeNodes };

      // 3. 在 items 数组中找到这个节点，并用新的 updatedItem 替换它
      const updateItemInTree = (nodes, targetId, newNode) => {
        for (let i = 0; i < nodes.length; i++) {
          if (nodes[i].id === targetId) {
            nodes[i] = newNode; // 直接替换该节点
            return true;
          }
          if (nodes[i].children) {
            const found = updateItemInTree(
              nodes[i].children,
              targetId,
              newNode
            );
            if (found) return true;
          }
        }
        return false;
      };

      // 4. 执行替换
      updateItemInTree(items.value, item.id, updatedItem);
    }

    // 5. 切换展开状态（你的原有逻辑）
    toggleFolderExpanded(item);
  }
};

// 懒加载子内容：根据文件夹节点，从 allFiles 中找出属于该文件夹的子文件/文件夹
const loadLazyChildrenForFolder = (folderItem) => {
  console.log("🔧 懒加载文件夹子内容，当前文件夹 path =", folderItem.path);

  // 假设 folderItem.path 是类似 "vue-vscode/src" 的字符串
  const folderPathPrefix = folderItem.fullPath + "/"; // 如 vue-vscode/src/
  console.log("🔍 筛选路径前缀：", folderPathPrefix);
  console.log(allFiles.value);

  // 从所有文件中筛选属于该文件夹下的文件
  const subItems = allFiles.value.filter((file) => {
    return file.webkitRelativePath.startsWith(folderPathPrefix);
  });

  console.log(`📂 找到 ${subItems.length} 个子项`);

  const childMap = new Map();

  subItems.forEach((file) => {
    console.log(1111);
    console.log(file);
    const relativePath = file.webkitRelativePath.slice(folderPathPrefix.length); // 去掉前缀，如 "App.vue" 或 "components/Button.vue"

    if (!relativePath) {
      console.warn("⚠️ 相对路径为空，可能路径匹配异常，跳过该文件：", file);
      return;
    }

    const pathParts = relativePath.split("/");

    const nodeName = pathParts[0]; // 如 App.vue、components

    if (!nodeName) {
      console.warn("⚠️ 解析出的节点名称为空，跳过该文件：", file);
      return;
    }

    const fileName = nodeName;
    const hasExtension = fileName.includes(".");
    const isDirectory = !hasExtension;
    const type = isDirectory ? "folder" : "file";
    const ext = isDirectory
      ? ""
      : fileName.split(".").pop()?.toLowerCase() || "";

    const childFullPath = folderPathPrefix + nodeName;

    if (!childMap.has(nodeName)) {
      const childNode = {
        name: nodeName,
        path: nodeName, // 或 full路径 childFullPath
        isDirectory: isDirectory,
        file: isDirectory ? null : file,
        type: type,
        ext: ext,
        children: [],
        expanded: false,
        fullPath: childFullPath,
        id: uuidv4(), // ✅ 每个节点唯一 ID，标准方式
      };
      childMap.set(nodeName, childNode);
    }
  });

  return Array.from(childMap.values());
};

const toggleFolderExpanded = (folderItem) => {
  // 递归查找并修改当前文件夹的 expanded 状态
  const updateExpanded = (items, targetId) => {
    for (const item of items) {
      if (item.id === targetId) {
        item.expanded = !item.expanded; // 切换展开状态
        return true; // 找到并修改成功
      }
      if (item.children) {
        const found = updateExpanded(item.children, targetId);
        if (found) return true;
      }
    }
    return false;
  };

  updateExpanded(items.value, folderItem.id);
};
// 双击逻辑（原来 onItemDoubleClick 的逻辑）
const onDoubleClick = (treeItem) => {
  console.log("双击:", treeItem);

  if (treeItem.type === "file" && treeItem.file) {
    // 双击文件：读取文件内容
    selectedFileName.value = treeItem.name;
    const file = treeItem.file;
    selectedId.value = treeItem.id;

    const reader = new FileReader();
    reader.readAsText(file);

    reader.onload = (event) => {
      selectedFileContent.value = event.target.result;
      emits("fileSelected", {
        name: selectedFileName.value,
        content: selectedFileContent.value,
        id: treeItem.id,
        selected: true,
      });
      //  可以在这里额外执行双击的操作
    };

    reader.onerror = () => {
      selectedFileContent.value = `无法读取文件：${reader.error.message}`;
    };
  } else if (treeItem.type === "folder") {
    // 双击文件夹：不执行任何操作（直接忽略）
    console.log("双击文件夹，不做任何操作");
  }
};

//复选框
const onItemChecked = (checkedItems) => {
  console.log("Checked12345678:", checkedItems);
};

const onBeforeItemDropped = (droppedItem, destinationNode) => {
  return new Promise((resolve) => {
    resolve(droppedItem !== destinationNode);
  });
};

const onItemExpanded = (expandedItem) => {
  console.log("Expanded123456:", expandedItem);
  const plainObject = JSON.parse(JSON.stringify(expandedItem));
  console.log(plainObject);

  // if (isDoubleClick.value) {
  //   // 如果是双击，则忽略单击
  //   isDoubleClick.value = false; //  重置 isDoubleClick
  //   clearTimeout(clickTimer.value); //  清除残留的定时器
  //   return;
  // }

  // // 300ms 后判断是否是双击
  // clickTimer.value = setTimeout(() => {
  //   if (!isDoubleClick.value) {
  //     // 300ms 内没有第二次点击，执行单击逻辑
  //     onSingleClick(plainObject);
  //   }
  //   clickTimer.value = null; //  清除定时器引用
  //   isDoubleClick.value = false; //  重置 isDoubleClick
  // }, 300);
};

defineExpose({
  selectFolder,
});
</script>

<style scoped lang="scss">
.file-tree {
  width: 100%;
  height: 100%;
  overflow-x: hidden; /* 隐藏水平滚动条 */
  overflow-y: auto; /* 保留垂直滚动 */
  background-color: #242526;
}

/* 新增树容器样式，用于限制宽度 */
.tree-container {
  margin-top: 20px;
  width: 100%;
  box-sizing: border-box;
  padding-right: 8px; /* 预留一点空间避免紧贴边缘 */
}

.open-btn {
  display: flex;
  flex-direction: row;
  justify-content: center;
  margin-top: 50px;
}

.selected-css {
  // background-color: pink;
  border: 1px solid #007fd4 !important;
}
</style>

<style lang="scss">
.tiny_horizontal_margin {
  height: 30px !important;
  // border:1px solid red;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}

.norem-tree {
  // 确保树组件宽度继承父容器
  width: 100%;
  .tree-item-node-parent {
    width: 100%;
  }

  // .tree-item-node {
  //   border:1px solid yellow;
  //   position: relative !important;
  // }

  .tree-view-item {
    // border: 1px solid yellow;
    position: relative !important;
    .chevron-right {
      display: inline-block;
      position: absolute;
      top: 11px;
      left: 6px;
      // border: 1px solid red;
      // width: 20px;
    }
  }

  .tree-item {
    // 限制每个树节点的最大宽度
    width: 100%;
    // width: 300px !important;
    // border: 1px solid green;
    .d-flex {
      display: flex !important;
      flex-direction: row !important;
      justify-content: flex-start !important;
      align-items: center !important;
      width: 100%;
      height: 30px;

      position: relative;
      .tree-node-name {
        // background-color: red;
        padding-left: 40px;
        border: 1px solid transparent;
      }

      // border:1px solid yellow;

      // 图标容器 - 固定宽度
      .item-prepend-icon {
        flex: 0 0 24px; /* 固定宽度，不伸缩 */
        margin-right: 6px;
      }

      .file-icon-border {
        width: 60px;
        height: 30px;
        padding-left: 10px;
        // margin-right: 6px;
        // background-color: blue;
        z-index: 1000;
        left: -15px;
        position: absolute;
        // border: 1px solid #ffffff;
        display: flex;
        flex-direction: row;
        justify-content: flex-end;
        align-items: center;
      }

      .norem-file-icon {
        width: 20px;
        height: 20px;
        // border: 1px solid red;
      }

      // 文件名容器 - 自适应剩余宽度
      span {
        display: inline-block;
        height: 30px;
        line-height: 30px;
        font-size: 16px;
        flex: 1; // 占满剩余空间
        min-width: 0; // 允许被压缩到比内容还小
        overflow: hidden; // 隐藏溢出部分
        white-space: nowrap; // 禁止换行，保证一行显示
        text-overflow: ellipsis; // 超出显示 ...
        // border: 1px solid red; // 可删除，调试用
        color: #b7b7b7;
      }
    }
  }
}

.vue3-tree-vue .guide-line {
  width: 12px;
  min-width: 12px !important;
  // border: 1px solid red;
}

.vue3-tree-vue .chevron-right {
  color: #b7b7b7;
}

.vue3-tree-vue .selected-tree-item {
  background: transparent;
}

//scroll
.file-tree-dark {
  border: 4px solid red;

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
