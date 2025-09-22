<template>
  <div class="editor-container">
    <div class="editor-split">
      <!-- 右上角主题切换按钮 -->
      <div class="theme-switcher">
        <button @click="toggleTheme" class="theme-btn" title="切换主题">
          {{ isDarkTheme ? "🌞 浅色" : "🌙 深色" }}
        </button>
      </div>
      <div ref="editorDom" class="code-editor"></div>
    </div>
  </div>
</template>

<script setup>
import { ref, watch, onMounted, onBeforeUnmount, onActivated } from "vue";
import beautify from "js-beautify";
import * as monaco from "monaco-editor";

// 🔧 Props 定义
const props = defineProps({
  modelValue: {
    type: String,
    default: '// 请输入代码\nconsole.log("Hello World!");',
  },
  language: {
    type: String,
    default: "javascript",
  },
  readOnly: {
    type: Boolean,
    default: false,
  },
  config: {
    type: Object,
    default: () => ({}),
  },
  // 可选：初始主题，可以是 'vs' | 'vs-dark' | 'hc-black' | 'my-custom-dark'
  initialTheme: {
    type: String,
    default: "my-custom-dark", // 默认深色，你可以改成 'my-custom-dark' 来直接用自定义背景色
  },
});

// 📤 Emits 定义
const emits = defineEmits([
  "update:modelValue",
  "change",
  "ready",
  "run",
  "error",
]);

// 🎯 Template Ref
const editorDom = ref(null);
let editorInstance = null;

// 📝 编辑器内容相关
const resultContent = ref("");
const isRunning = ref(false);

// 🌈 主题相关状态
const isDarkTheme = ref(false);
const currentTheme = ref(props.initialTheme);

// 🎨 主题切换方法（浅色/深色切换，内置主题）
const toggleTheme = () => {
  if (currentTheme.value === "vs") {
    // currentTheme.value = "vs-dark";
    currentTheme.value = "my-custom-dark"; //切换到自定义
    isDarkTheme.value = true;
  } else {
    currentTheme.value = "vs";
    isDarkTheme.value = false;
  }
  if (editorInstance) {
    monaco.editor.setTheme(currentTheme.value);
  }
};

// 🛠️ 手动设置主题 my-custom-dark
const setTheme = (themeName) => {
  currentTheme.value = themeName;
  if (themeName === "vs-dark") {
    isDarkTheme.value = true;
  } else if (themeName === "vs") {
    isDarkTheme.value = false;
  } else {
    // 自定义主题，可以根据名字自行判断是否深色
    isDarkTheme.value = true; // 假设其他都是深色
  }
  if (editorInstance) {
    monaco.editor.setTheme(themeName);
  }
};

// 🧹 代码格式化
const format = () => {
  const formatted = beautify(editorInstance.getValue(), {
    indent_size: 4,
    preserve_newlines: true,
  });
  editorInstance?.setValue(formatted);
};

// 👁️ 聚焦编辑器
const focus = () => {
  editorInstance?.focus();
};

// ▶️ 运行代码逻辑（略，与你原来一样）
const runCode = async () => {
  if (isRunning.value) return;

  const code = editorInstance.getValue();
  if (!code.trim()) {
    resultContent.value = "请输入代码后再运行";
    return;
  }

  if (props.language === "javascript") {
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
    if (!window.pyodide) {
      resultContent.value = "Pyodide 未加载，请先加载 Pyodide 运行时";
      return;
    }
    try {
      isRunning.value = true;
      resultContent.value = "运行中...";
      emits("run", code);

      window.pyodide.globals.set("print", (...args) => {
        const msg = args.map((arg) => window.pyodide.repr(arg)).join(" ");
        resultContent.value += msg + "\n";
      });

      const result = await window.pyodide.runPythonAsync(code);
      if (result !== undefined && result !== null) {
        resultContent.value += "\n返回结果: " + window.pyodide.repr(result);
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

// 🧹 清空运行结果
const clearResult = () => {
  resultContent.value = "";
};

// 🧩 暴露方法给父组件
defineExpose({
  format,
  focus,
  runCode,
  clearResult,
  setTheme,
  toggleTheme,
});

// --------------------------
// 生命周期 & 监听
// --------------------------

// 初始化编辑器
// 初始化编辑器
onMounted(() => {
  if (!editorDom.value) return;

  editorInstance = monaco.editor.create(editorDom.value, {
    value: props.modelValue,
    language: props.language,
    automaticLayout: true,
    parameterHints: { enabled: true },
    minimap: { enabled: true },
    wrappingStrategy: "advanced",
    scrollBeyondLastLine: false,
    fontSize: 14,
    readOnly: props.readOnly,
    stopTrustedEvents: false,
    ...props.config,
  });

  // ======================
  // ✅ 正确设置主题（包括自定义主题 my-custom-dark）
  // ======================

  const theme = currentTheme.value; // 可能是 'vs', 'vs-dark', 'my-custom-dark'

  // 如果是自定义主题 'my-custom-dark'，需要先定义它
  if (theme === "my-custom-dark") {
    monaco.editor.defineTheme("my-custom-dark", {
      base: "vs-dark",
      inherit: true,
      rules: [],
      colors: {
        // ✅ 重点：这就是你想要的背景色 #292a2b
        "editor.background": "#292a2b",
        "editor.foreground": "#cccccc",
        "editorLineNumber.foreground": "#858585",
        "editorCursor.foreground": "#ffffff",
        "editor.selectionBackground": "#3e3e3e",
      },
    });
  }

  // 然后再设置主题（无论是内置还是自定义）
  monaco.editor.setTheme(theme);

  // 更新 isDarkTheme 状态（可选，用于 UI 按钮显示）
  if (theme === "vs-dark" || theme === "my-custom-dark") {
    isDarkTheme.value = true;
  } else {
    isDarkTheme.value = false;
  }

  // 监听编辑器内容变化
  editorInstance.onDidChangeModelContent(() => {
    const value = editorInstance.getValue();
    emits("update:modelValue", value);
    emits("change", value);
  });

  // 通知编辑器已 ready
  emits("ready", editorInstance);
});

// 激活时聚焦
onActivated(() => {
  editorInstance?.focus();
});

// 销毁时清理
onBeforeUnmount(() => {
  editorInstance?.dispose();
});

// 监听 modelValue 变化
watch(
  () => props.modelValue,
  (newValue) => {
    const currentValue = editorInstance?.getValue();
    if (newValue !== currentValue) {
      editorInstance?.setValue(newValue);
    }
  }
);

// 监听 readOnly
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
</script>

<style lang="scss" scoped>
.editor-container {
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
}

.editor-split {
  height: 100%;
  width: 100%;
  position: relative;
  box-sizing: border-box;
}

.code-editor {
  height: 100%;
  width: 100%;
  box-sizing: border-box;
}

.theme-switcher {
  position: absolute;
  top: 10px;
  right: 10px;
  z-index: 10;
}

.theme-btn {
  background: rgba(255, 255, 255, 0.9);
  border: 1px solid #ccc;
  border-radius: 4px;
  padding: 4px 8px;
  font-size: 12px;
  cursor: pointer;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);

  &:hover {
    background: rgba(255, 255, 255, 1);
  }
}
</style>
