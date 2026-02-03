<template>
  <div :class="['sidebar-container', { 'collapsed': isCollapsed }]">
    <div class="sidebar-header">
      <img src="~@/assets/sidebar.png" class="sidebar-toggle-icon" @click="$emit('toggle')" />
    </div>
    
    <div class="sidebar-content">
      <div v-for="(item, index) in fixedItems" :key="'fixed-' + index"
        :class="['sidebar-item', { 'active': activeId === 'f' + index }]"
        @click="handleFixedClick(index)">
        <img :src="item.iconPath" class="item-icon" />
        <span v-show="!isCollapsed" class="item-text">{{ item.name }}</span>
      </div>

      <div v-show="!isCollapsed && historyItems.length > 0" class="divider"></div>

      <div v-show="!isCollapsed" class="history-list">
        <div v-for="item in historyItems" :key="item.id"
          :class="['sidebar-item', 'history-item', { 'active': activeId === item.id }]"
          @click="activeId = item.id">
          
          <template v-if="editingId === item.id">
            <input 
              :ref="'renameInput-' + item.id"
              v-model="editName"
              class="rename-input"
              @blur="saveEdit(item)"
              @keyup.enter="saveEdit(item)"
              @keyup.esc="cancelEdit"
              @click.stop
            />
          </template>
          <template v-else>
            <span class="item-text history-text">{{ item.name }}</span>
          </template>
          
          <img v-if="editingId !== item.id"
            src="~@/assets/more.png" 
            class="more-icon" 
            @click.stop="toggleMenu(item.id)" 
          />

          <div v-if="openMenuId === item.id" class="popup-menu" @click.stop>
            <div class="menu-option" @click="handleRename(item)">
              <span>重命名</span>
            </div>
            <div class="menu-option delete" @click="handleDelete(item.id)">
              <span>删除</span>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'SideBar',
  props: { isCollapsed: Boolean },
  data() {
    return {
      activeId: 'f0',
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
  mounted() {
    document.addEventListener('click', this.closeMenu);
  },
  beforeDestroy() {
    document.removeEventListener('click', this.closeMenu);
  },
  methods: {
    handleFixedClick(index) {
      this.activeId = 'f' + index;
      if (index === 0) this.createNewTask();
      this.closeMenu();
    },
    createNewTask() {
      const newId = 'h-' + Date.now();
      this.historyItems.unshift({ id: newId, name: `新对话 ${this.historyItems.length + 1}` });
      this.activeId = newId;
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
        const inputRef = this.$refs['renameInput-' + item.id];
        if (inputRef && inputRef[0]) inputRef[0].focus();
      });
    },
    saveEdit(item) {
      if (!this.editingId) return;
      if (this.editName.trim()) item.name = this.editName.trim();
      this.cancelEdit();
    },
    cancelEdit() {
      this.editingId = null;
      this.editName = '';
    },
    handleDelete(id) {
      this.historyItems = this.historyItems.filter(item => item.id !== id);
      if (this.activeId === id) this.activeId = 'f0';
      this.closeMenu();
    }
  }
}
</script>

<style scoped>
/* 保持原有布局样式 */
.sidebar-container { width: 300px; height: 100vh; background: #ebebeb; display: flex; flex-direction: column; border-right: 1px solid #e5e7eb; transition: width 0.3s ease; overflow: hidden; }
.sidebar-container.collapsed { width: 64px; }
.sidebar-header { height: 56px; display: flex; align-items: center; justify-content: flex-end; padding: 0 16px; flex-shrink: 0; }
.sidebar-container.collapsed .sidebar-header { justify-content: center; padding: 0; }
.sidebar-toggle-icon { width: 24px; height: 24px; cursor: pointer; }
.sidebar-content { flex: 1; padding: 12px 0; display: flex; flex-direction: column; align-items: center; }

/* Item 样式 */
.sidebar-item { position: relative; width: 284px; height: 36px; border-radius: 8px; display: flex; align-items: center; padding: 0 12px; box-sizing: border-box; cursor: pointer; margin-bottom: 4px; transition: background 0.2s; }
.sidebar-item.active { background: #e0e0e0; }
.sidebar-item:hover:not(.active) { background: rgba(0, 0, 0, 0.05); }

.item-icon { width: 20px; height: 20px; flex-shrink: 0; margin-right: 8px; }
.item-text { width: 244px; height: 20px; font-size: 14px; font-family: "PingFang SC", sans-serif; color: rgba(0, 0, 0, 0.75); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.history-text { padding-left: 28px; transition: width 0.2s; }
.history-item:hover .history-text { width: 200px; }

/* 重命名输入框样式 */
.rename-input { margin-left: 28px; width: 200px; height: 24px; font-size: 14px; font-family: "PingFang SC", sans-serif; border: 1px solid #42b983; border-radius: 4px; outline: none; padding: 0 4px; background: #fff; color: #333; }

/* More 图标 */
.more-icon { position: absolute; right: 8px; top: 8px; width: 20px; height: 20px; opacity: 0; transition: opacity 0.2s; z-index: 10; }
.sidebar-item:hover .more-icon { opacity: 1; }
.sidebar-item .more-icon[style*="opacity: 1"] { opacity: 1; } 

/* 弹窗菜单样式 */
.popup-menu {
  position: absolute; top: 32px; right: 8px; width: 100px; background: #ffffff; border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15); padding: 4px; z-index: 100; border: 1px solid #eee;
}
.menu-option { display: flex; align-items: center; padding: 6px 12px; font-size: 13px; color: #333; border-radius: 4px; transition: background 0.2s; cursor: pointer; }
.menu-option:hover { background: #f5f5f5; }
.menu-option.delete span { color: #ff4d4f; }
.divider { width: 260px; height: 1px; background: rgba(0,0,0,0.05); margin: 12px 0; }
.history-list { width: 100%; display: flex; flex-direction: column; align-items: center; }
</style>