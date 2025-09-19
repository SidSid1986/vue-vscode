<template>
  <div class="terminal-container">
    <!-- 主题切换按钮 -->
    <div class="theme-controls">
      <button
        @click="switchTheme('dark')"
        :class="{ active: currentTheme === 'dark' }"
      >
        深色模式
      </button>
      <button
        @click="switchTheme('light')"
        :class="{ active: currentTheme === 'light' }"
      >
        浅色模式
      </button>
    </div>
    <div ref="terminalRef" class="terminal"></div>
  </div>
</template>

<script setup>
import { onMounted, ref, onUnmounted, getCurrentInstance } from "vue";
import { Terminal } from "xterm";
import { FitAddon } from "xterm-addon-fit";
import "xterm/css/xterm.css";

// 全局WebSocket实例
const instance = getCurrentInstance();
const $ws = instance.appContext.config.globalProperties.$XPack_WebSocket;

// 终端核心实例与状态
const terminalRef = ref(null);
let terminal = null;
const fitAddon = new FitAddon();
let inputBuffer = "";
let isWsConnected = false;
let isCommandSent = false;
let isLoading = ref(false);

// 1. 命令历史记录
const commandHistory = ref([]); // 存储历史命令
const historyIndex = ref(-1); // 当前历史索引

// 2. 路径自动补全（模拟数据）
const mockPaths = [
  "/d/sid/vue-vscode-git",
  "/d/sid/vue-vscode-git/src",
  "/d/sid/vue-vscode-git/public",
  "/d/sid/vue-vscode-git/package.json",
  "/d/sid/docs",
  "/d/sid/tests",
  "/d/sid/vue-vscode-git/src/components",
  "/d/sid/vue-vscode-git/src/views",
];

// 4. 终端主题（v5.x 配置）
const currentTheme = ref("dark");
const themes = {
  dark: {
    background: "#1e1e1e",
    foreground: "#d4d4d4",
    cursor: "#ffffff",
    selectionBackground: "rgba(255, 255, 255, 0.2)",
  },
  light: {
    background: "#ffffff",
    foreground: "#333333",
    cursor: "#000000",
    selectionBackground: "rgba(0, 0, 0, 0.2)",
  },
};

// 路径和提示符配置
const promptPrefix = "DELL@DESKTOP-V48IIM0F MINGW64";
const currentPath = ref("/d/sid/vue-vscode-git (main)");
const ANSI_COLOR = {
  PATH_FOREGROUND: "\x1b[32m", // 路径绿色
  RESET: "\x1b[0m", // 重置样式
  ERROR: "\x1b[31m", // 错误红色
  INFO: "\x1b[36m", // 信息青色
};

// 初始化终端
const initTerminal = () => {
  // 销毁旧实例
  if (terminal) {
    terminal.dispose();
  }

  // 创建新终端实例（v5.x 配置）
  terminal = new Terminal({
    fontSize: 14,
    fontFamily: "Consolas, Monaco, monospace",
    cursorBlink: true,
    theme: themes[currentTheme.value], // 初始主题
    unicodeActive: true,
    scrollback: 1000, // 滚动历史长度
  });

  // 加载插件
  terminal.loadAddon(fitAddon);
  terminal.open(terminalRef.value);
  fitAddon.fit();

  // 初始显示
  showPathLine();
  showPromptLine();

  // 绑定输入事件
  bindTerminalInputEvents();
};

// 绑定终端输入事件
const bindTerminalInputEvents = () => {
  terminal.onData((data) => {
    // 上箭头：调用上一条历史命令
    if (data === "\x1b[A") {
      if (
        commandHistory.value.length > 0 &&
        historyIndex.value < commandHistory.value.length - 1
      ) {
        historyIndex.value++;
        inputBuffer =
          commandHistory.value[
            commandHistory.value.length - 1 - historyIndex.value
          ] || "";
        updateInputDisplay();
      }
      return;
    }

    // 下箭头：调用下一条历史命令
    if (data === "\x1b[B") {
      if (historyIndex.value > 0) {
        historyIndex.value--;
        inputBuffer =
          commandHistory.value[
            commandHistory.value.length - 1 - historyIndex.value
          ] || "";
      } else {
        historyIndex.value = -1;
        inputBuffer = "";
      }
      updateInputDisplay();
      return;
    }

    // Tab键：路径自动补全
    if (data === "\t") {
      handlePathCompletion();
      return;
    }

    // 退格键：删除最后一个字符
    if (data.charCodeAt(0) === 127) {
      if (inputBuffer.length > 0) {
        inputBuffer = inputBuffer.slice(0, -1);
        updateInputDisplay();
      }
      return;
    }

    // 回车键：执行命令
    if (data === "\r") {
      const command = inputBuffer.trim();
      if (command && isWsConnected) {
        // 记录历史命令（去重）
        if (!commandHistory.value.includes(command)) {
          commandHistory.value.push(command);
        }
        historyIndex.value = -1; // 重置历史索引

        terminal.writeln("");
        isCommandSent = true;
        isLoading.value = true;
        terminal.write(`${ANSI_COLOR.INFO}⌛ 执行中...${ANSI_COLOR.RESET}`);

        // 发送命令到后端
        $ws.Send({
          eventFlag: "terminalCommand",
          data: command,
        });
      } else if (!command) {
        // 空命令：重新显示路径和提示符
        terminal.writeln("");
        showPathLine();
        showPromptLine();
      }
      inputBuffer = "";
      return;
    }

    // 可打印字符：添加到输入缓冲区
    if (isPrintable(data)) {
      inputBuffer += data;
      updateInputDisplay();
    }
  });
};

// 更新输入显示
const updateInputDisplay = () => {
  clearCurrentLine();
  terminal.write(`$ ${inputBuffer}`);
};

// 路径自动补全逻辑
const handlePathCompletion = () => {
  // 仅处理cd命令的路径补全
  if (!inputBuffer.startsWith("cd ")) return;

  const pathPart = inputBuffer.slice(3).trim();
  if (!pathPart) return;

  // 匹配可能的补全路径
  const candidates = mockPaths.filter(
    (path) => path.startsWith(pathPart) && path !== pathPart
  );

  if (candidates.length === 1) {
    // 唯一匹配：直接补全
    inputBuffer = `cd ${candidates[0]}`;
    updateInputDisplay();
  } else if (candidates.length > 1) {
    // 多个匹配：显示所有候选路径
    terminal.writeln("");
    terminal.write(candidates.join("  "));
    terminal.writeln("");
    showPromptLine();
    terminal.write(inputBuffer); // 重新显示当前输入
  }
};

// 初始化WebSocket连接
const initWs = () => {
  $ws
    .init("")
    .then((code) => {
      if (code === 200) {
        isWsConnected = true;
        terminal.writeln("✅ 终端WS连接成功");
        showPathLine();
        showPromptLine();

        // 监听命令执行结果
        $ws.EventAdd("terminalResult", (result) => {
          if (isLoading.value) {
            clearCurrentLine();
            isLoading.value = false;
          }

          // 6. 错误命令提示
          if (
            result.includes("not found") ||
            result.includes("不是内部或外部命令")
          ) {
            const command = inputBuffer.trim().split(" ")[0];
            const suggestions = getCommandSuggestions(command);
            terminal.writeln(`${ANSI_COLOR.ERROR}❌ 命令不存在: ${command}`);
            if (suggestions.length > 0) {
              terminal.writeln(
                `💡 可能的命令: ${suggestions.join(", ")}${ANSI_COLOR.RESET}`
              );
            }
          } else {
            // 正常结果处理
            if (isCommandSent) {
              terminal.writeln(result);
              isCommandSent = false;
            } else {
              terminal.write(result);
            }
          }

          // 显示新的路径和提示符
          showPathLine();
          showPromptLine();
        });

        // 监听连接关闭
        $ws.EventAdd("connectionClose", () => {
          isWsConnected = false;
          terminal.writeln("\r\n❌ 终端WS连接已断开");
        });
      }
    })
    .catch((error) => {
      terminal.writeln(`\r\n❌ 终端WS连接失败：${error.message}`);
    });
};

// 获取命令建议（错误提示辅助）
const getCommandSuggestions = (wrongCommand) => {
  const commonCommands = [
    "node",
    "npm",
    "git",
    "cd",
    "ls",
    "dir",
    "python",
    "pip",
    "yarn",
  ];
  return commonCommands.filter(
    (cmd) => cmd.includes(wrongCommand) || wrongCommand.includes(cmd)
  );
};

// 4. 切换主题（v5.x 正确方式）
const switchTheme = (theme) => {
  if (currentTheme.value === theme || !terminal) return;

  currentTheme.value = theme;
  // 直接修改options.theme（v5.x官方推荐）
  terminal.options.theme = { ...themes[theme] };
  // 强制刷新终端以应用新主题
  terminal.refresh(0, terminal.rows - 1);
};

// 5. 窗口大小自适应
const handleResize = () => {
  if (terminal) {
    fitAddon.fit();
  }
};

// 辅助方法：显示路径行
const showPathLine = () => {
  const coloredPath = `${ANSI_COLOR.PATH_FOREGROUND}${promptPrefix} ${currentPath.value}${ANSI_COLOR.RESET}`;
  terminal.writeln(coloredPath);
};

// 辅助方法：显示提示符
const showPromptLine = () => {
  terminal.write(`$ `);
};

// 辅助方法：清除当前行
const clearCurrentLine = () => {
  terminal.write("\x1b[2K"); // 清除当前行
  terminal.write("\x1b[0G"); // 光标移到行首
};

// 辅助方法：判断是否为可打印字符
const isPrintable = (data) => {
  const code = data.charCodeAt(0);
  return (code >= 32 && code <= 126) || code > 127;
};

// 生命周期
onMounted(() => {
  initTerminal();
  initWs();
  window.addEventListener("resize", handleResize);
});

onUnmounted(() => {
  if (terminal) terminal.dispose();
  if ($ws && isWsConnected) {
    $ws.Close();
    isWsConnected = false;
  }
  window.removeEventListener("resize", handleResize);
});
</script>

<style scoped>
.terminal-container {
  width: 500px;

  border: 2px solid red;
  border-radius: 4px;
  overflow: hidden;
  position: relative;
}

/* 主题切换按钮样式 */
.theme-controls {
  position: absolute;
  top: 8px;
  right: 8px;
  z-index: 10;
  display: flex;
  gap: 8px;
}

.theme-controls button {
  padding: 4px 8px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 12px;
  background: #555;
  color: white;
  transition: background 0.2s;
}

.theme-controls button.active {
  background: #0078d7;
}

.theme-controls button:hover:not(.active) {
  background: #666;
}

/* 终端样式 */
.terminal {
  width: 100%;
  height: 100%;
}
</style>
