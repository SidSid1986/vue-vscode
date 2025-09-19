<template>
  <div class="terminal-container">
    <div ref="terminalRef" class="terminal"></div>
  </div>
</template>

<script setup>
import { onMounted, ref, onUnmounted, getCurrentInstance } from "vue";
import { Terminal } from "xterm";
import { FitAddon } from "xterm-addon-fit";
import "xterm/css/xterm.css";

const instance = getCurrentInstance();
const $ws = instance.appContext.config.globalProperties.$XPack_WebSocket;

const terminalRef = ref(null);
let terminal = null;
const fitAddon = new FitAddon();
let inputBuffer = "";
let isWsConnected = false;
let isCommandSent = false; // 标记命令是否已发送（用于控制换行）

const promptPrefix = "DELL@DESKTOP-V48IIM0F MINGW64";
const currentPath = ref("/d/sid/vue-vscode-git (main)");

onMounted(() => {
  initTerminal();
  initWs();
});

onUnmounted(() => {
  if (terminal) terminal.dispose();
  if ($ws && isWsConnected) {
    $ws.Close();
    isWsConnected = false;
  }
});

const initTerminal = () => {
  terminal = new Terminal({
    fontSize: 14,
    fontFamily: "Consolas, Monaco, monospace",
    cursorBlink: true,
    theme: { background: "#1e1e1e", foreground: "#d4d4d4", cursor: "#ffffff" },
    unicodeActive: true,
  });
  terminal.loadAddon(fitAddon);
  terminal.open(terminalRef.value);
  fitAddon.fit();
  showPathLine();
  showPromptLine();

  terminal.onData((data) => {
    const keyCode = data.charCodeAt(0);

    // 处理退格键
    if (keyCode === 127) {
      if (inputBuffer.length > 0) {
        inputBuffer = inputBuffer.slice(0, -1);
        clearCurrentLine();
        terminal.write(`$ ${inputBuffer}`);
      }
      return;
    }

    // 处理回车键
    if (data === "\r") {
      const command = inputBuffer.trim();
      if (command && isWsConnected) {
        // 关键：输入命令后先换行，与结果分离
        terminal.writeln(""); 
        isCommandSent = true; // 标记命令已发送，等待结果
        $ws.Send({
          eventFlag: "terminalCommand",
          data: command,
        });
      } else if (!command) {
        terminal.writeln("");
        showPathLine();
        showPromptLine();
      }
      inputBuffer = "";
      return;
    }

    // 处理可打印字符
    if (isPrintable(data)) {
      inputBuffer += data;
      clearCurrentLine();
      terminal.write(`$ ${inputBuffer}`);
    }
  });
};

const showPathLine = () => {
  terminal.writeln(`${promptPrefix} ${currentPath.value}`);
};

const showPromptLine = () => {
  terminal.write(`$ `);
};

const initWs = () => {
  $ws
    .init("")
    .then((code) => {
      if (code === 200) {
        isWsConnected = true;
        terminal.writeln("✅ 终端WS连接成功");
        showPathLine();
        showPromptLine();
        $ws.EventAdd("terminalResult", (result) => {
          // 关键：如果命令已发送，先换行再显示结果
          if (isCommandSent) {
            terminal.writeln(result); // 结果单独换行
            isCommandSent = false; // 重置标记
          } else {
            terminal.write(result);
          }
          showPathLine();
          showPromptLine();
        });
        $ws.EventAdd("connectionClose", () => {
          isWsConnected = false;
          terminal.writeln("\r\n❌ 终端WS连接已断开");
        });
      }
    })
    .catch((error) => {
      terminal.writeln(`\r\n❌ 终端WS连接失败：${error.message}`);
      console.error("WS初始化错误详情：", error);
    });
};

const clearCurrentLine = () => {
  terminal.write("\x1b[2K");
  terminal.write("\x1b[0G");
};

const isPrintable = (data) => {
  const code = data.charCodeAt(0);
  return (code >= 32 && code <= 126) || code > 127;
};
</script>

<style scoped>
.terminal-container {
  width: 100%;
  height: 400px;
  border: 1px solid #333;
  border-radius: 4px;
  overflow: hidden;
}
.terminal {
  width: 100%;
  height: 100%;
}
</style>
