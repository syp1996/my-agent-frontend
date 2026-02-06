<template>
  <div class="chat-container">
    <div class="chat-header"></div> 
    
    <div class="chat-box" ref="chatBox">
      <div class="messages-wrapper">
        <div v-for="(msg, index) in messages" :key="index" :class="['message', msg.role]">
          <div v-if="msg.role === 'ai' && (msg.content || msg.hasThought)" class="ai-label-container">
            <span class="ai-label">Agent</span>
          </div>

          <div class="message-content" v-if="msg.content || msg.hasThought">
            
            <div v-if="msg.role === 'ai' && msg.hasThought" class="thought-process">
              <div class="thought-header" @click="msg.isThoughtExpanded = !msg.isThoughtExpanded">
                <span v-if="!msg.isDoneThinking" class="thinking-spinner">🔄</span>
                <span v-else class="thinking-icon">💡</span>
                <span class="thought-title">
                  {{ msg.isDoneThinking ? '思考已完成' : '深度思考中...' }}
                </span>
                <span class="toggle-arrow">{{ msg.isThoughtExpanded ? '▼' : '▶' }}</span>
              </div>

              <div v-if="msg.isThoughtExpanded" class="thought-body">
                <div v-for="(step, sIndex) in msg.steps" :key="sIndex" class="step-item">
                  <span class="step-icon">{{ step.status === 'loading' ? '⏳' : '✅' }}</span>
                  <span class="step-text">{{ step.title }}</span>
                </div>

                <div v-if="msg.thoughts" class="thought-text">
                  {{ msg.thoughts }}
                </div>
              </div>
            </div>

            <div 
              v-if="msg.role === 'ai'" 
              class="markdown-body" 
              v-html="renderMarkdown(msg.content)"
            ></div>
            <span v-else style="white-space: pre-wrap;">{{ msg.content }}</span>
          </div>
        </div>
        
        <div v-if="isLoadingHistory" class="loading-tip">正在拉取历史记录...</div>
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
import DOMPurify from 'dompurify';
import hljs from 'highlight.js';
import 'highlight.js/styles/github.css';
import MarkdownIt from 'markdown-it';

const md = new MarkdownIt({
  html: true,
  linkify: true,
  typographer: true,
  highlight: function (str, lang) {
    if (lang && hljs.getLanguage(lang)) {
      try {
        return '<pre class="hljs"><code>' +
               hljs.highlight(str, { language: lang, ignoreIllegals: true }).value +
               '</code></pre>';
      } catch (__) { console.error(__); }
    }
    return '<pre class="hljs"><code>' + md.utils.escapeHtml(str) + '</code></pre>';
  }
});

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
    renderMarkdown(content) {
      if (!content) return '';
      const rawHtml = md.render(content);
      return DOMPurify.sanitize(rawHtml);
    },
      async fetchHistory(id) {
      this.isLoadingHistory = true;
      this.messages = [];
      this.userInput = ''; 
      try {
        const res = await fetch(`http://localhost:8000/threads/${id}/history`);
        if (res.ok) {
          const historyData = await res.json();
            const rawMessages = historyData.history || [];
          this.messages = rawMessages.map(msg => ({
            ...msg,
            role: msg.role === 'assistant' ? 'ai' : msg.role
          }));
          this.scrollToBottom();
        }
      } catch (e) { 
        console.error("获取历史记录失败:", e); 
      } finally { 
        this.isLoadingHistory = false; 
      }
    },
    async sendMessage() {
      if (!this.userInput || this.isStreaming) return;
      const isFirstMessage = this.messages.length === 0;
      const query = this.userInput;
      this.messages.push({ role: 'user', content: query });
      this.userInput = ''; 
      this.isStreaming = true;
      
      const aiMessage = { 
        role: 'ai', 
        content: '', 
        thoughts: '',          
        steps: [],             
        hasThought: false,     
        isDoneThinking: false, 
        isThoughtExpanded: true 
      };
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
        while (this.isStreaming) {
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
                if (dataStr === '[DONE]') {
                   // 结束信号
                } else { 
                   try { eventData = JSON.parse(dataStr); } catch (e) { console.error(e); } 
                }
              }
            });

            if (eventType === 'thought' && eventData) {
              aiMessage.thoughts += eventData.content;
              aiMessage.hasThought = true;
              this.scrollToBottom();
            }
            else if (eventType === 'step' && eventData) {
              aiMessage.hasThought = true; 
              const lastStep = aiMessage.steps[aiMessage.steps.length - 1];
              if (lastStep && lastStep.status === 'loading' && eventData.status === 'done') {
                 lastStep.status = 'done';
                 if (eventData.title) lastStep.title = eventData.title;
              } else {
                 aiMessage.steps.push(eventData);
              }
              this.scrollToBottom();
            }
            else if (eventType === 'message' && eventData) {
              if (!aiMessage.isDoneThinking) {
                aiMessage.isDoneThinking = true;
                aiMessage.isThoughtExpanded = false; 
              }
              aiMessage.content += eventData.content;
              this.scrollToBottom();
            }
            else if (eventType === 'title_generated' && eventData) {
              this.$emit('title-generated', {
                thread_id: eventData.thread_id,
                title: eventData.title
              });
            }
          }
        }
      } catch (error) { aiMessage.content += "\n[连接失败]"; }
      finally { 
        this.isStreaming = false; 
        if (aiMessage) aiMessage.isDoneThinking = true;
        if (isFirstMessage) {
          this.$emit('first-message-sent');
        }
      }
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
/* 样式保持不变 */
.chat-container { display: flex; flex-direction: column; height: 100%; width: 100%; position: relative; background: transparent; }
.chat-header { height: 56px; }
.chat-box { flex: 1; overflow-y: auto; padding: 20px 40px; padding-bottom: 240px; display: flex; flex-direction: column; align-items: center; z-index: 1; outline: none; }
.messages-wrapper { width: 80%; max-width: 733px; display: flex; flex-direction: column; }
.bottom-mask { position: absolute; bottom: 0; left: 0; width: 100%; height: 200px; background: #f8f8f7; z-index: 8; }
.input-section { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); width: 80%; max-width: 733px; z-index: 10; transition: all 0.5s cubic-bezier(0.2, 1, 0.3, 1); outline: none; }
.input-section.at-bottom { top: calc(100% - 104px); }
.custom-input-box { position: relative; width: 100%; height: 164px; background: #ffffff; border-radius: 20px; box-shadow: 0 4px 24px rgba(0,0,0,0.08); padding: 20px; box-sizing: border-box; outline: none; }
textarea { width: 100%; height: calc(100% - 30px); border: none; outline: none; resize: none; font-size: 16px; font-family: 'PingFang SC', sans-serif; color: #333; background: transparent; box-shadow: none; }
.play-icon { position: absolute; bottom: 20px; right: 20px; width: 24px; height: 24px; cursor: pointer; }
.message { margin-bottom: 24px; display: flex; width: 100%; }
.message.user { justify-content: flex-end; }
.user .message-content { max-width: 85%; background: #f0f0f0; color: #333; padding: 12px 18px; border-radius: 18px; }
.message.ai { flex-direction: column; align-items: flex-start; } 
.ai .message-content { width: 100%; max-width: 100%; background: #fff; border: 1px solid #f0f0f0; box-shadow: 0 2px 8px rgba(0,0,0,0.02); padding: 12px 18px; border-radius: 18px; border-top-left-radius: 0; line-height: 1.6; font-size: 15px; box-sizing: border-box; }
.ai-label-container { width: 100%; display: flex; justify-content: flex-start; padding-left: 0; box-sizing: border-box; margin-bottom: -1px; position: relative; z-index: 2; }
.ai-label { color: #666; font-size: 14px; padding: 12px 12px; border-radius: 8px 8px 0 0; border-bottom: none; font-weight: 500; }
.thought-process { background-color: #f7f7f8; border-radius: 8px; margin-bottom: 12px; border: 1px solid #e5e5e5; overflow: hidden; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; }
.thought-header { display: flex; align-items: center; padding: 8px 12px; cursor: pointer; background: #f0f0f0; font-size: 13px; color: #666; user-select: none; transition: background 0.2s; }
.thought-header:hover { background: #e8e8e8; }
.thinking-spinner { animation: spin 1s linear infinite; margin-right: 8px; font-size: 12px; }
.thinking-icon { margin-right: 8px; font-size: 12px; }
.thought-title { flex: 1; font-weight: 500; }
.toggle-arrow { font-size: 10px; color: #999; margin-left: 8px; }
.thought-body { padding: 10px 12px; border-top: 1px solid #e5e5e5; background: #fff; }
.step-item { display: flex; align-items: flex-start; margin-bottom: 8px; font-size: 13px; color: #444; line-height: 1.5; }
.step-icon { margin-right: 8px; min-width: 16px; text-align: center; }
.thought-text { margin-top: 8px; padding-top: 8px; border-top: 1px dashed #eee; color: #888; font-size: 13px; line-height: 1.6; white-space: pre-wrap; }
@keyframes spin { from { transform: rotate(0deg); } to { transform: rotate(360deg); } }
.markdown-body { word-break: break-word; }
.markdown-body >>> p { margin: 0 0 10px 0; }
.markdown-body >>> p:last-child { margin-bottom: 0; }
.markdown-body >>> code { background-color: rgba(175, 184, 193, 0.2); padding: 0.2em 0.4em; border-radius: 6px; font-family: monospace; }
.markdown-body >>> pre code { background-color: transparent; padding: 0; }
.markdown-body >>> .hljs { padding: 12px; border-radius: 8px; margin: 10px 0; overflow-x: auto; background: #f6f8fa; }
.markdown-body >>> ul, .markdown-body >>> ol { padding-left: 2em; margin-bottom: 10px; }
.loading-tip { text-align: center; color: #bbb; font-size: 13px; padding: 20px; }
.scroll-anchor { height: 1px; width: 100%; flex-shrink: 0; }
</style>