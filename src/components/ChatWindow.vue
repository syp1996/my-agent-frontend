<template>
  <div class="chat-container">
    <div class="chat-header"></div> 
    
    <div class="chat-box" ref="chatBox">
      <div v-for="(msg, index) in messages" :key="index" :class="['message', msg.role]">
        <div class="message-content">
          <span style="white-space: pre-wrap;">{{ msg.content }}</span>
        </div>
      </div>
      
      <div v-if="isLoadingHistory" class="loading-tip">
        正在拉取历史记录...
      </div>
      
      <div v-if="currentAgent" class="agent-status">
        <span class="status-dot"></span>
        ⚡ {{ currentAgent }}...
      </div>
    </div>

    <div class="input-section">
      <div class="custom-input-box">
        <textarea 
          v-model="userInput" 
          @keyup.enter.exact.prevent="sendMessage" 
          placeholder="问问我关于杭州地铁的一切..." 
          :disabled="isStreaming"
        ></textarea>
        <img src="~@/assets/play.png" class="play-icon" @click="sendMessage" />
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
      try {
        const res = await fetch(`http://localhost:8000/threads/${id}/history`);
        if (res.ok) {
          this.messages = await res.json();
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
        const container = this.$refs.chatBox;
        if (container) container.scrollTop = container.scrollHeight;
      });
    }
  }
};
</script>

<style scoped>
.chat-container { display: flex; flex-direction: column; height: 100%; width: 100%; position: relative; background: transparent; }
.chat-header { height: 56px; }
.chat-box { flex: 1; overflow-y: auto; padding: 20px 40px; }
.input-section { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); width: 80%; max-width: 733px; z-index: 10; transition: all 0.3s ease; }
.custom-input-box { position: relative; width: 100%; height: 164px; background: #ffffff; border-radius: 20px; box-shadow: 0 4px 24px rgba(0,0,0,0.08); padding: 20px; box-sizing: border-box; }
textarea { width: 100%; height: calc(100% - 30px); border: none; outline: none; resize: none; font-size: 16px; font-family: 'PingFang SC', sans-serif; color: #333; background: transparent; }
.play-icon { position: absolute; bottom: 20px; right: 20px; width: 24px; height: 24px; cursor: pointer; transition: opacity 0.2s; }
.message { margin-bottom: 24px; display: flex; }
.message.user { justify-content: flex-end; }
.message.ai { justify-content: flex-start; }
.message-content { max-width: 70%; padding: 12px 16px; border-radius: 12px; line-height: 1.6; }
.user .message-content { background: #e0e0e0; color: #333; }
.ai .message-content { background: #fff; border: 1px solid #eee; }
.agent-status { align-self: center; background: rgba(232, 245, 233, 0.8); color: #2e7d32; padding: 4px 12px; border-radius: 20px; font-size: 12px; margin-bottom: 10px; display: flex; align-items: center; }
.status-dot { width: 8px; height: 8px; background: #2e7d32; border-radius: 50%; margin-right: 6px; }
.loading-tip { text-align: center; color: #999; font-size: 12px; padding: 10px; }
</style>