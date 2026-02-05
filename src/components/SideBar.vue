<template>
  <div :class="['sidebar-container', { 'collapsed': isCollapsed }]">
    <div class="sidebar-header">
      <img src="~@/assets/sidebar.png" class="sidebar-toggle-icon" @click="$emit('toggle')" />
    </div>
    
    <div class="sidebar-content">
      <div 
        v-for="(item, index) in fixedItems" 
        :key="'fixed-' + index"
        :class="['sidebar-item', { 'active': activeId === 'f' + index }]"
        @click="handleFixedClick(index)"
      >
        <img :src="item.iconPath" class="item-icon" />
        <span v-show="!isCollapsed" class="item-text">{{ item.name }}</span>
      </div>

      <div v-show="!isCollapsed && historyItems.length > 0" class="divider"></div>

      <div v-show="!isCollapsed" class="history-list">
        <div 
          v-for="item in historyItems" 
          :key="item.id"
          :class="['sidebar-item', 'history-item', { 'active': activeId === item.id }]"
          @click="handleHistoryClick(item)"
        >
          <template v-if="editingId === item.id">
            <input 
              :ref="'renameInput-' + item.id"
              v-model="editName"
              class="rename-input"
              @blur="saveEdit(item)"
              @keyup.enter="saveEdit(item)"
              @click.stop
            />
          </template>
          <template v-else>
            <span class="item-text history-text" :title="item.name">{{ item.name }}</span>
          </template>
          
          <img v-if="editingId !== item.id" src="~@/assets/more.png" class="more-icon" @click.stop="toggleMenu(item.id)" />

          <div v-if="openMenuId === item.id" class="popup-menu" @click.stop>
            <div class="menu-option" @click="handleRename(item)"><span>重命名</span></div>
            <div class="menu-option delete" @click="handleDelete(item.id)"><span>删除</span></div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'SideBar',
  props: { 
    isCollapsed: Boolean, 
    currentThreadId: String 
  },
  data() {
    return {
      openMenuId: null,
      editingId: null,
      editName: '',
      fixedItems: [
        { name: '新建任务', iconPath: require('@/assets/new.png') },
        { name: '搜索', iconPath: require('@/assets/search.png') },
        { name: '库', iconPath: require('@/assets/library.png') }
      ],
      historyItems: []
    };
  },
  computed: {
    activeId() {
      if (this.historyItems.some(item => item.id === this.currentThreadId)) return this.currentThreadId;
      // 如果是新创建的 thread 但还没存入数据库，也高亮“新建任务”
      if (this.currentThreadId && this.currentThreadId.startsWith('thread_')) return 'f0';
      return null;
    }
  },
  mounted() {
    this.fetchThreads();
    document.addEventListener('click', this.closeMenu);
  },
  beforeDestroy() { 
    document.removeEventListener('click', this.closeMenu); 
  },
  methods: {
    async fetchThreads() {
      try {
        const res = await fetch('http://localhost:8000/threads');
        if (res.ok) {
          const data = await res.json();
          // 【核心修复】后端返回格式是 { "threads": ["id1", "id2"] }
          // 我们将字符串数组转换为对象数组供模板渲染
          if (data && data.threads) {
            this.historyItems = data.threads.map(id => ({
              id: id,
              name: id // 暂时用 ID 作为名称展示
            }));
          }
        }
      } catch (e) { 
        console.error("获取线程列表失败:", e); 
      }
    },
    handleHistoryClick(item) { 
      this.$emit('select-chat', item.id); 
    },
    handleFixedClick(index) {
      if (index === 0) {
        // 新建任务：生成临时 ID 并通知父组件
        const newId = 'thread_' + Date.now();
        this.$emit('select-chat', newId);
      }
    },
    async saveEdit(item) {
      if (!this.editingId) return;
      const newName = this.editName.trim();
      if (newName && newName !== item.name) {
        try {
          await fetch(`http://localhost:8000/threads/${item.id}/rename`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ title: newName })
          });
          item.name = newName;
        } catch (e) { console.error(e); }
      }
      this.cancelEdit();
    },
    async handleDelete(id) {
      if (!confirm("确定要删除此对话吗？")) return;
      try {
        await fetch(`http://localhost:8000/threads/${id}`, { method: 'DELETE' });
        this.historyItems = this.historyItems.filter(item => item.id !== id);
        if (this.currentThreadId === id) this.handleFixedClick(0);
      } catch (e) { console.error(e); }
      this.closeMenu();
    },
    toggleMenu(id) { 
      this.openMenuId = this.openMenuId === id ? null : id; 
    },
    closeMenu() { 
      this.openMenuId = null; 
    },
    handleRename(item) {
      this.closeMenu();
      this.editingId = item.id;
      this.editName = item.name;
      this.$nextTick(() => {
        const input = this.$refs['renameInput-' + item.id];
        if (input && input[0]) input[0].focus();
      });
    },
    cancelEdit() { 
      this.editingId = null; 
      this.editName = ''; 
    }
  }
}
</script>

<style scoped>
.sidebar-container { width: 300px; height: 100vh; background: #ebebeb; display: flex; flex-direction: column; border-right: 1px solid #e5e7eb; transition: width 0.3s ease; overflow: hidden; }
.sidebar-container.collapsed { width: 64px; }
.sidebar-header { height: 56px; display: flex; align-items: center; justify-content: flex-end; padding: 0 16px; box-sizing: border-box; }
.sidebar-toggle-icon { width: 24px; height: 24px; cursor: pointer; }
.sidebar-content { flex: 1; padding: 12px 0; display: flex; flex-direction: column; align-items: center; }
.sidebar-item { position: relative; width: 284px; height: 36px; border-radius: 8px; display: flex; align-items: center; padding: 0 12px; box-sizing: border-box; cursor: pointer; margin-bottom: 4px; transition: background 0.2s; }
.sidebar-item.active { background: #e0e0e0; }
.sidebar-item:hover:not(.active) { background: rgba(0, 0, 0, 0.05); }
.item-icon { width: 20px; height: 20px; flex-shrink: 0; margin-right: 8px; }
.item-text { width: 244px; font-size: 14px; color: rgba(0, 0, 0, 0.75); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.history-text { padding-left: 28px; }
.rename-input { margin-left: 28px; width: 200px; height: 24px; border: 1px solid #42b983; border-radius: 4px; outline: none; padding: 0 4px; }
.more-icon { position: absolute; right: 8px; width: 20px; opacity: 0; transition: opacity 0.2s ease; }
.sidebar-item:hover .more-icon { opacity: 1; }
.popup-menu { position: absolute; top: 32px; right: 8px; width: 100px; background: #fff; border-radius: 8px; box-shadow: 0 4px 12px rgba(0,0,0,0.15); padding: 4px; z-index: 100; border: 1px solid #eee; }
.menu-option { padding: 6px 12px; font-size: 13px; cursor: pointer; border-radius: 4px; }
.menu-option:hover { background: #f5f5f5; }
.menu-option.delete span { color: #ff4d4f; }
.divider { width: 260px; height: 1px; background: rgba(0,0,0,0.1); margin: 12px 0; }
.history-list { width: 100%; overflow-y: auto; flex: 1; }
</style>