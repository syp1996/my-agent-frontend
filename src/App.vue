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
      // 1. 初始值设为空，由 mounted 逻辑接管
      currentThreadId: ''
    }
  },
  created() {
    // 2. 在组件创建时初始化 ID
    this.initThreadId();
  },
  mounted() {
    // 3. 监听浏览器前进/后退，确保 URL 变化时页面同步
    window.addEventListener('hashchange', this.syncIdFromHash);
  },
  beforeDestroy() {
    window.removeEventListener('hashchange', this.syncIdFromHash);
  },
  methods: {
    initThreadId() {
      // 优先级：URL Hash > localStorage > 新生成
      const hashId = window.location.hash.replace('#/', '');
      const localId = localStorage.getItem('last_thread_id');
      
      if (hashId && hashId.startsWith('thread_')) {
        this.currentThreadId = hashId;
      } else if (localId) {
        this.currentThreadId = localId;
        window.location.hash = `/${localId}`;
      } else {
        const newId = 'thread_' + Date.now();
        this.currentThreadId = newId;
        this.updateBrowserState(newId);
      }
    },
    syncIdFromHash() {
      const hashId = window.location.hash.replace('#/', '');
      if (hashId && hashId !== this.currentThreadId) {
        this.currentThreadId = hashId;
      }
    },
    updateCurrentThread(id) {
      this.currentThreadId = id;
      this.updateBrowserState(id);
    },
    updateBrowserState(id) {
      // 更新 URL Hash 和本地存储
      window.location.hash = `/${id}`;
      localStorage.setItem('last_thread_id', id);
    },
    refreshSidebar() {
      setTimeout(() => {
        if (this.$refs.sidebar) {
          this.$refs.sidebar.fetchThreads();
        }
      }, 500); 
    },
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