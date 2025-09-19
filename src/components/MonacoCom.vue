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
  width: 70vw;
  height: 100vh;
  display: flex;
  flex-direction: row;
  border: 1px solid #e0e0e0;
  border-radius: 4px;
}

.editor-split {
  border: 1px solid blue;
  height: 100vh;
  width: 100%;
}

.code-editor {
  border: 1px solid blue;
  height: 100vh;
  width: 100%;
}
</style>
