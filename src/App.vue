<!--
 * @Author: Yunpeng Shi y.shi27@newcastle.ac.uk
 * @Date: 2026-01-22 08:55:12
 * @LastEditors: Yunpeng Shi y.shi27@newcastle.ac.uk
 * @LastEditTime: 2026-01-22 08:58:48
 * @FilePath: /my-agent-frontend/src/App.vue
 * @Description: 这是默认设置,请设置`customMade`, 打开koroFileHeader查看配置 进行设置: https://github.com/OBKoro1/koro1FileHeader/wiki/%E9%85%8D%E7%BD%AE
-->
<template>
  <div id="app">
    <ChatWindow />
  </div>
</template>

<script>
import ChatWindow from './components/ChatWindow.vue';

export default {
  name: 'App',
  components: {
    ChatWindow
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
