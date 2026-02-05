<template>
  <div class="chat-container">
    <div class="chat-header"></div> 
    
    <div class="chat-box" ref="chatBox">
      <div class="messages-wrapper">
        <div v-for="(msg, index) in messages" :key="index" :class="['message', msg.role]">
          <div class="message-content">
            <span style="white-space: pre-wrap;">{{ msg.content }}</span>
          </div>
        </div>
        
        <div v-if="isLoadingHistory" class="loading-tip">正在拉取历史记录...</div>
        
        <div v-if="currentAgent" class="agent-status">
          <span class="status-dot"></span>
          ⚡ {{ currentAgent }}...
        </div>
      </div>
      <div class="scroll-anchor"></div>
    </div>

    <div v-if="messages.length > 0 || isLoadingHistory" class="bottom-mask"></div>

    <div :class="['input-section', { 'at-bottom': messages.length > 0 || isLoadingHistory }]">
      <div class="custom-input-box">
        <textarea 
          v-model="userInput" 
          @keyup.enter.exact.prevent="sendMessage" 
          placeholder="问问我关于杭州地铁的一切..." 
          :disabled="isStreaming"
        ></textarea>
        <img src="~@/assets/play.png" class="play-icon" @click="sendMessage" alt="send" />
      </div>
    </div>
  </div>
</template>

<script>
export default {
  props: {
    threadId: { type: String, required: true }
  },
  data() {
    return {
      userInput: '',
      messages: [], 
      isStreaming: false,
      isLoadingHistory: false,
      currentAgent: '',
    };
  },
  watch: {
    threadId: {
      immediate: true,
      handler(newVal) {
        if (newVal) this.fetchHistory(newVal);
      }
    }
  },
  methods: {
    async fetchHistory(id) {
      this.isLoadingHistory = true;
      this.messages = [];
      this.userInput = ''; 
      try {
        const res = await fetch(`http://localhost:8000/threads/${id}/history`);
        if (res.ok) {
          const historyData = await res.json();
          this.messages = historyData;
          this.scrollToBottom();
        }
      } catch (e) { console.error(e); }
      finally { this.isLoadingHistory = false; }
    },
    async sendMessage() {
      if (!this.userInput || this.isStreaming) return;
      const query = this.userInput;
      this.messages.push({ role: 'user', content: query });
      this.userInput = ''; 
      this.isStreaming = true;
      this.currentAgent = 'Supervisor (调度中)';
      
      const aiMessage = { role: 'ai', content: '' };
      this.messages.push(aiMessage);
      this.scrollToBottom();

      try {
        const response = await fetch('http://localhost:8000/chat/stream', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ query, thread_id: this.threadId })
        });
        const reader = response.body.getReader();
        const decoder = new TextDecoder();
        let buffer = '';
        /* eslint-disable-next-line no-constant-condition */
        while (true) {
          const { done, value } = await reader.read();
          if (done) break;
          buffer += decoder.decode(value, { stream: true });
          const events = buffer.split('\n\n');
          buffer = events.pop();
          for (const eventStr of events) {
            if (!eventStr.trim()) continue;
            const lines = eventStr.split('\n');
            let eventType = null;
            let eventData = null;
            lines.forEach(line => {
              if (line.startsWith('event: ')) eventType = line.substring(7).trim();
              else if (line.startsWith('data: ')) {
                const dataStr = line.substring(6).trim();
                if (dataStr === '[DONE]') this.currentAgent = '';
                else { try { eventData = JSON.parse(dataStr); } catch (e) { console.error(e); } }
              }
            });
            if (eventType === 'agent_start' && eventData) { this.currentAgent = eventData.agent; this.scrollToBottom(); }
            else if (eventType === 'message' && eventData) { aiMessage.content += eventData.content; this.scrollToBottom(); }
          }
        }
      } catch (error) { aiMessage.content += "\n[连接失败]"; }
      finally { this.isStreaming = false; this.currentAgent = ''; }
    },
    scrollToBottom() {
      this.$nextTick(() => {
        setTimeout(() => {
          const container = this.$refs.chatBox;
          if (container) container.scrollTo({ top: container.scrollHeight, behavior: 'smooth' });
        }, 100);
      });
    }
  }
};
</script>

<style scoped>
.chat-container { display: flex; flex-direction: column; height: 100%; width: 100%; position: relative; background: transparent; }
.chat-header { height: 56px; }

.chat-box { 
  flex: 1; 
  overflow-y: auto; 
  padding: 20px 40px; 
  padding-bottom: 240px; 
  display: flex;
  flex-direction: column;
  align-items: center;
  z-index: 1;
  outline: none;
}

/* 消息包装层，其宽度固定为 733px 与输入框一致 */
.messages-wrapper { width: 80%; max-width: 733px; display: flex; flex-direction: column; }

.bottom-mask {
  position: absolute;
  bottom: 0;
  left: 0;
  width: 100%;
  height: 200px;
  background: #f8f8f7; 
  z-index: 8;
}

.input-section { 
  position: absolute; 
  top: 50%; 
  left: 50%; 
  transform: translate(-50%, -50%); 
  width: 80%; 
  max-width: 733px; 
  z-index: 10; 
  transition: all 0.5s cubic-bezier(0.2, 1, 0.3, 1);
  outline: none;
}

.input-section.at-bottom { top: calc(100% - 104px); }

.custom-input-box { 
  position: relative; 
  width: 100%; 
  height: 164px; 
  background: #ffffff; 
  border-radius: 20px; 
  box-shadow: 0 4px 24px rgba(0,0,0,0.08); 
  padding: 20px; 
  box-sizing: border-box; 
  outline: none;
}

textarea { 
  width: 100%; 
  height: calc(100% - 30px); 
  border: none; 
  outline: none; 
  resize: none; 
  font-size: 16px; 
  font-family: 'PingFang SC', sans-serif; 
  color: #333; 
  background: transparent; 
  box-shadow: none;
}

.play-icon { position: absolute; bottom: 20px; right: 20px; width: 24px; height: 24px; cursor: pointer; }

/* 消息样式调整 */
.message { margin-bottom: 24px; display: flex; width: 100%; }

/* 用户消息保持右对齐 */
.message.user { justify-content: flex-end; }
.user .message-content { max-width: 85%; background: #f0f0f0; color: #333; }

/* AI 消息调整为居中对齐，且内容块填满 733px 宽度以实现对齐效果 */
.message.ai { justify-content: center; } 
.ai .message-content { 
  width: 100%; /* 填满 messages-wrapper 的宽度 */
  max-width: 100%; 
  background: #fff; 
  border: 1px solid #f0f0f0; 
  box-shadow: 0 2px 8px rgba(0,0,0,0.02); 
  padding: 12px 18px; 
  border-radius: 18px; 
  line-height: 1.6; 
  font-size: 15px;
}

/* 状态提示也调整为居中 */
.agent-status { 
  align-self: center; 
  background: rgba(232, 245, 233, 0.9); 
  color: #2e7d32; 
  padding: 6px 14px; 
  border-radius: 20px; 
  font-size: 12px; 
  margin-bottom: 20px; 
  display: flex; 
  align-items: center; 
}
.status-dot { width: 8px; height: 8px; background: #2e7d32; border-radius: 50%; margin-right: 8px; }
.loading-tip { text-align: center; color: #bbb; font-size: 13px; padding: 20px; }
.scroll-anchor { height: 1px; width: 100%; flex-shrink: 0; }
</style>