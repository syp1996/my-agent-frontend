<template>
  <div id="app">
    <SideBar 
      ref="sidebar" 
      :is-collapsed="isCollapsed" 
      :current-thread-id="currentThreadId"
      @toggle="isCollapsed = !isCollapsed" 
      @select-chat="updateCurrentThread" 
    />
    
    <div class="main-content">
      <ChatWindow 
        :key="currentThreadId" 
        :thread-id="currentThreadId" 
        @first-message-sent="refreshSidebar"
        @title-generated="onTitleGenerated"
      />
    </div>
  </div>
</template>

<script>
import ChatWindow from './components/ChatWindow.vue';
import SideBar from './components/SideBar.vue';

export default {
  name: 'App',
  components: { SideBar, ChatWindow },
  data() {
    return {
      isCollapsed: false,
      currentThreadId: 'thread_' + Date.now()
    }
  },
  methods: {
    updateCurrentThread(id) {
      this.currentThreadId = id;
    },
    refreshSidebar() {
      setTimeout(() => {
        if (this.$refs.sidebar) {
          this.$refs.sidebar.fetchThreads();
        }
      }, 500); 
    },
    // --- 核心修改：接收标题并更新侧边栏 ---
    onTitleGenerated(payload) {
      if (this.$refs.sidebar) {
        this.$refs.sidebar.updateItemTitle(payload);
      }
    }
  }
}
</script>

<style>
/* 保持原有样式不变 */
html, body { margin: 0; padding: 0; height: 100%; background: #f8f8f7; overflow: hidden; }
#app { display: flex; height: 100vh; font-family: 'PingFang SC', Arial, sans-serif; }
.main-content { flex: 1; height: 100%; overflow: hidden; transition: all 0.3s ease; display: flex; flex-direction: column; }
</style>