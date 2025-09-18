<template>
  <div class="terminal-container">
    <div ref="terminalRef" class="terminal"></div>
  </div>
</template>

<script setup>
import { onMounted, ref, onUnmounted } from 'vue';
import { Terminal } from 'xterm';
import { FitAddon } from 'xterm-addon-fit';
import 'xterm/css/xterm.css'; // 引入默认样式

const terminalRef = ref(null);
let terminal = null;
const fitAddon = new FitAddon();

onMounted(() => {
  // 初始化终端
  terminal = new Terminal({
    fontSize: 14,
    fontFamily: 'Consolas, Monaco, monospace',
    cursorBlink: true, // 光标闪烁
    theme: {
      background: '#1e1e1e', // 终端背景（类似VS Code深色主题）
      foreground: '#d4d4d4',
      cursor: '#ffffff'
    }
  });

  // 加载插件并挂载到DOM
  terminal.loadAddon(fitAddon);
  terminal.open(terminalRef.value);
  fitAddon.fit(); // 自适应容器尺寸

  // 初始欢迎信息
  terminal.writeln('Welcome to the terminal!\r\n');

  // 监听用户输入（前端模拟，实际需对接后端）
  terminal.onData((data) => {
    // 示例：回显输入内容（实际应发送到后端执行命令）
    if (data === '\r') { // 回车
      terminal.writeln('\r\n$ '); // 模拟命令提示符
    } else {
      terminal.write(data); // 实时回显输入
    }
  });
});

onUnmounted(() => {
  // 销毁终端实例
  if (terminal) {
    terminal.dispose();
  }
});
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
