<template>
  <div id="app">
    <SideBar 
      :is-collapsed="isCollapsed" 
      @toggle="isCollapsed = !isCollapsed" 
    />
    
    <div class="main-content">
      <ChatWindow />
    </div>
  </div>
</template>

<script>
import ChatWindow from './components/ChatWindow.vue';
import SideBar from './components/SideBar.vue';

export default {
  name: 'App',
  components: {
    SideBar,
    ChatWindow
  },
  data() {
    return {
      isCollapsed: false // 默认为展开状态
    };
  }
}
</script>

<style>
/* 保持全局背景和布局 */
html, body {
  margin: 0;
  padding: 0;
  height: 100%;
  background: #f8f8f7;
}

#app {
  display: flex;
  height: 100vh;
}

.main-content {
  flex: 1;
  transition: all 0.3s ease; /* 让右侧区域跟随侧边栏平滑移动 */
}
</style>