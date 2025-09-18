<template>
  <div class="editor-container">
    <div class="editor-split">
      <!-- 左侧代码编辑区 -->
      <div ref="editorDom" class="code-editor"></div>
    </div>
  </div>
</template>

<script setup>
import { ref, watch, onMounted, onBeforeUnmount, onActivated } from "vue";
import beautify from "js-beautify";
import * as monaco from "monaco-editor";
// import 'monaco-languages/release/esm/monaco-languages.js'

// Props 定义
const props = defineProps({
  modelValue: {
    type: String,
    default: '// 请输入代码\nconsole.log("Hello World!");',
  },
  language: {
    type: String,
    default: "javascript", // 默认支持JavaScript
  },

  readOnly: {
    type: Boolean,
    default: false,
  },
  config: {
    type: Object,
    default: () => ({}),
  },
});

// Emits 定义
const emits = defineEmits([
  "update:modelValue",
  "change",
  "ready",
  "run",
  "error",
]);

//
const pyodide = ref(null);
const pyodideReady = ref(false);

// 模板 Ref
const editorDom = ref(null);
let editorInstance = null;

// 运行结果相关
const resultContent = ref("");
const isRunning = ref(false);

// 监听 modelValue 变化，同步到编辑器
watch(
  () => props.modelValue,
  (newValue) => {
    const currentValue = editorInstance?.getValue();
    if (newValue !== currentValue) {
      editorInstance?.setValue(newValue);
    }
  }
);

watch(
  () => pyodideReady.value,
  (isReady) => {
    if (isReady) {
      console.log("Pyodide 加载完成，可以运行 Python 代码了");
      // 可选：加载完成后自动执行一段测试代码
      // runCode();
    }
  }
);

// 监听 readOnly 变化，更新编辑器配置
watch(
  () => props.readOnly,
  (readOnly) => {
    editorInstance?.updateOptions({
      readOnly,
    });
  }
);

// 监听语言变化
watch(
  () => props.language,
  (language) => {
    if (editorInstance) {
      monaco.editor.setModelLanguage(editorInstance.getModel(), language);
    }
  }
);

// 组件挂载后初始化 Monaco Editor
onMounted(() => {
  // 创建编辑器实例
  editorInstance = monaco.editor.create(editorDom.value, {
    value: props.modelValue,
    language: props.language,
    automaticLayout: true,
    parameterHints: {
      enabled: true,
    },
    minimap: {
      enabled: true,
    },
    wrappingStrategy: "advanced",
    scrollBeyondLastLine: false,
    fontSize: 14,
    readOnly: props.readOnly,
    stopTrustedEvents: false,
    ...props.config,
  });

  // 监听编辑器内容变化，向父组件同步
  editorInstance.onDidChangeModelContent(() => {
    const value = editorInstance.getValue();
    emits("update:modelValue", value);
    emits("change", value);
  });

  // 通知父组件编辑器已 ready
  emits("ready", editorInstance);
});

// Tab 页激活时聚焦编辑器
onActivated(() => {
  editorInstance?.focus();
});

// 组件卸载前销毁编辑器，防止内存泄漏
onBeforeUnmount(() => {
  editorInstance?.dispose();
});

// 运行代码js
// const runCode = () => {
//   if (isRunning.value) return;

//   const code = editorInstance.getValue();
//   if (!code.trim()) {
//     resultContent.value = "请输入代码后再运行";
//     return;
//   }

//   try {
//     isRunning.value = true;
//     resultContent.value = "运行中...";
//     emits("run", code);

//     // 这里使用沙箱环境执行代码，避免污染全局
//     // 注意：在生产环境中执行用户代码需要更严格的安全措施
//     const output = [];
//     const console = {
//       log: (...args) =>
//         output.push(
//           ...args.map((arg) =>
//             typeof arg === "object" ? JSON.stringify(arg, null, 2) : String(arg)
//           )
//         ),
//       error: (...args) => output.push(...args.map((arg) => `Error: ${arg}`)),
//     };

//     // 使用IIFE执行代码，捕获输出
//     const result = (function (console) {
//       try {
//         return eval(code); // 注意：eval有安全风险，生产环境需谨慎
//       } catch (e) {
//         console.error(e.message);
//         return null;
//       }
//     })(console);

//     if (result !== undefined) {
//       output.push("返回结果:", result);
//     }

//     resultContent.value = output.join("\n");
//   } catch (error) {
//     resultContent.value = `执行错误: ${error.message}`;
//     emits("error", error);
//   } finally {
//     isRunning.value = false;
//   }
// };

// 运行代码python
const runCode = async () => {
  if (isRunning.value) return;

  const code = editorInstance.getValue();
  if (!code.trim()) {
    resultContent.value = "请输入代码后再运行";
    return;
  }

  if (props.language === "javascript") {
    // 保留原 JS 执行逻辑（不变）
    try {
      isRunning.value = true;
      resultContent.value = "运行中...";
      emits("run", code);

      const output = [];
      const console = {
        log: (...args) =>
          output.push(
            ...args.map((arg) =>
              typeof arg === "object"
                ? JSON.stringify(arg, null, 2)
                : String(arg)
            )
          ),
        error: (...args) => output.push(...args.map((arg) => `Error: ${arg}`)),
      };

      const result = (function (console) {
        try {
          return eval(code);
        } catch (e) {
          console.error(e.message);
          return null;
        }
      })(console);

      if (result !== undefined) {
        output.push("返回结果:", result);
      }

      resultContent.value = output.join("\n");
    } catch (error) {
      resultContent.value = `执行错误: ${error.message}`;
      emits("error", error);
    } finally {
      isRunning.value = false;
    }
  } else if (props.language === "python") {
    // Python 执行逻辑（依赖 Pyodide，已在 onMounted 中加载）
    if (!pyodideReady.value) {
      resultContent.value = "Pyodide 正在加载，请稍后重试";
      return;
    }
    try {
      isRunning.value = true;
      resultContent.value = "运行中...";
      // 重定向 Python 的 print 到结果区（解决原逻辑无输出问题）
      pyodide.value.globals.set("print", (...args) => {
        const msg = args.map((arg) => pyodide.value.repr(arg)).join(" ");
        resultContent.value += msg + "\n";
      });
      // 执行 Python 代码
      const result = await pyodide.value.runPythonAsync(code);
      // 补充返回结果（若有）
      if (result !== undefined && result !== null) {
        resultContent.value += "\n返回结果: " + pyodide.value.repr(result);
      }
    } catch (error) {
      resultContent.value = `执行错误: ${error.message}`;
      emits("error", error);
    } finally {
      isRunning.value = false;
    }
  } else {
    resultContent.value = "不支持该语言运行";
  }
};
// 清空结果
const clearResult = () => {
  resultContent.value = "";
};

// 暴露格式化方法
const format = () => {
  const formatted = beautify(editorInstance.getValue(), {
    indent_size: 4,
    preserve_newlines: true,
  });
  editorInstance?.setValue(formatted);
};

// 暴露聚焦方法
const focus = () => {
  editorInstance?.focus();
};

// 向父组件暴露方法
defineExpose({
  format,
  focus,
  runCode,
  clearResult,
});
</script>

<style lang="scss" scoped>
.editor-container {
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  border: 1px solid #e0e0e0;
  border-radius: 4px;
  overflow: hidden;
}

.editor-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px;
  background-color: #f5f5f5;
  border-bottom: 1px solid #e0e0e0;

  h3 {
    margin: 0;
    font-size: 20px;
    font-weight: 600;
    color: #333;
  }
}

.run-button {
  background-color: #42b983;
  color: white;
  border: none;
  padding: 10px;
  border-radius: 4px;
  cursor: pointer;
  display: flex;
  align-items: center;
  gap: 10px;
  transition: background-color 0.2s;

  &:hover {
    background-color: #359e75;
  }

  &:disabled {
    background-color: #cccccc;
    cursor: not-allowed;
  }
}

.editor-split {
  display: flex;
  flex: 1;
  width: 100%;
  height: 0; /* 让flex决定高度 */
  min-height: 0;
}

.code-editor {
  flex: 1;
  border-right: 1px solid #e0e0e0;
}

.result-panel {
  flex: 1;
  display: flex;
  flex-direction: column;
  background-color: #fafafa;
}

.result-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px;
  border-bottom: 1px solid #e0e0e0;
}

.clear-button {
  background-color: transparent;
  color: #666;
  border: 1px solid #ddd;
  padding: 10px;
  border-radius: 4px;
  cursor: pointer;
  transition: all 0.2s;

  &:hover {
    background-color: #f0f0f0;
  }

  &:disabled {
    color: #ccc;
    cursor: not-allowed;
  }
}

.result-content {
  flex: 1;
  padding: 10px;
  overflow: auto;
  position: relative;
  /* 关键：允许文本选择 */
  user-select: text !important;
  -webkit-user-select: text !important; /* 兼容 Safari */
  -moz-user-select: text !important; /* 兼容 Firefox */
}

.output {
  margin: 0;
  white-space: pre-wrap;
  word-wrap: break-word;
  font-family: monospace;
  font-size: 14px;
  line-height: 1.5;
  color: #333;
  /* 确保文本可选中 */
  user-select: text !important;
  -webkit-user-select: text !important;
}

.empty-result {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  color: #999;
  font-style: italic;
}

// 图标样式
.icon-run::before {
  content: "▶";
}
</style>
