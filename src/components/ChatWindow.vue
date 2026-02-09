<template>
  <div class="chat-container">
    <div class="chat-header"></div> 
    
    <div class="chat-box" ref="chatBox">
      <div class="messages-wrapper">
        <div v-for="(msg, index) in messages" :key="index" :class="['message', msg.role]">
          <div v-if="msg.role === 'ai'" class="ai-label-container">
            <span class="ai-label">Agent</span>
          </div>

          <div class="message-content" v-if="msg.role === 'ai' || msg.content || msg.hasThought">
            
            <div v-if="msg.role === 'ai' && !msg.hasThought && !msg.content" class="preparing-state">
              <span class="dot-flashing"></span>
              <span class="preparing-text">正在接入系统...</span>
            </div>

            <template v-else>
              <div v-if="msg.role === 'ai' && msg.hasThought" :class="['thought-process', { 'thinking-active': !msg.isDoneThinking }]">
                <div class="thought-header" @click="msg.isThoughtExpanded = !msg.isThoughtExpanded">
                  <div class="status-indicator">
                    <span v-if="!msg.isDoneThinking" class="thinking-spinner-modern"></span>
                    <span v-else class="thinking-icon-done">✨</span>
                  </div>
                  <div class="status-text">
                    <span class="status-title">{{ msg.isDoneThinking ? '思考已完成' : '正在思考' }}</span>
                    <span class="status-timer" v-if="msg.thinkingDuration > 0">{{ msg.thinkingDuration }}s</span>
                  </div>
                  <span :class="['toggle-arrow', { 'is-expanded': msg.isThoughtExpanded }]">
                    <svg viewBox="0 0 24 24" width="16" height="16" stroke="currentColor" stroke-width="2" fill="none"><polyline points="6 9 12 15 18 9"></polyline></svg>
                  </span>
                </div>

                <div v-if="msg.isThoughtExpanded" class="thought-body">
                  <div class="timeline-container">
                    <div v-for="(item, tIndex) in msg.timeline" :key="tIndex" class="thought-node">
                      <div class="node-header">
                        <div class="node-icon">
                          <span v-if="item.status === 'loading'" class="spinner-small"></span>
                          <span v-else class="icon-success">✓</span>
                        </div>
                        <span class="node-title">{{ item.title }}</span>
                      </div>
                      <div v-if="item.content" :class="['node-body', 'markdown-body', { 'is-streaming': !msg.isDoneThinking && tIndex === msg.timeline.length - 1 }]" v-html="renderMarkdown(item.content)"></div>
                    </div>
                  </div>
                </div>
              </div>

              <div v-if="msg.role === 'ai'" class="markdown-body" v-html="renderMarkdown(msg.content)"></div>
              <span v-else style="white-space: pre-wrap;">{{ msg.content }}</span>
            </template>
          </div>
        </div>
        <div v-if="isLoadingHistory" class="loading-tip">正在同步历史记录...</div>
      </div>
      <div class="scroll-anchor"></div>
    </div>

    <div :class="['input-section', { 'at-bottom': messages.length > 0 || isLoadingHistory }]">
      <div class="custom-input-box">
        <textarea v-model="userInput" @keyup.enter.exact.prevent="sendMessage" placeholder="描述您的问题..." :disabled="isStreaming"></textarea>
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
  highlight: (str, lang) => {
    if (lang && hljs.getLanguage(lang)) {
      try {
        return hljs.highlight(str, { language: lang }).value;
      } catch (err) {
        console.error('Highlight error:', err);
      }
    }
    return md.utils.escapeHtml(str);
  }
});

export default {
  props: { threadId: { type: String, required: true } },
  data() { return { userInput: '', messages: [], isStreaming: false, isLoadingHistory: false, timerInterval: null }; },
  watch: { threadId: { immediate: true, handler(v) { if (v) this.fetchHistory(v); } } },
  methods: {
    renderMarkdown(c) {
      if (!c) return '';
      // 深度清洗残留标签
      const cleaned = c.replace(/^(Content:|Title:.*?\n)/gim, '').trim();
      return DOMPurify.sanitize(md.render(cleaned));
    },
    parseThoughts(raw) {
      const nodes = [];
      const regex = /Title:\s*(.*?)\s*\n\s*Content:\s*([\s\S]*?)(?=\n\s*Title:|$)/g;
      let m;
      while ((m = regex.exec(raw)) !== null) {
        nodes.push({ title: m[1].trim(), content: m[2].trim(), status: 'done' });
      }
      return nodes.length ? nodes : [{ title: '思维链', content: raw, status: 'done' }];
    },
    async fetchHistory(id) {
      this.isLoadingHistory = true;
      this.messages = [];
      try {
        const res = await fetch(`http://localhost:8000/threads/${id}/history`);
        const data = await res.json();
        this.messages = (data.history || []).map(msg => {
          const isAi = msg.role === 'assistant';
          let timeline = isAi && msg.thoughts ? this.parseThoughts(msg.thoughts) : [];
          if (isAi && msg.steps) msg.steps.forEach(s => timeline.push({ ...s, content: '' }));
          return { ...msg, role: isAi ? 'ai' : msg.role, timeline, hasThought: timeline.length > 0, isDoneThinking: true, isThoughtExpanded: false, thinkingDuration: 0 };
        });
        this.scrollToBottom();
      } catch (err) {
        console.error('Fetch history failed:', err);
      } finally {
        this.isLoadingHistory = false;
      }
    },
    async sendMessage() {
      if (!this.userInput || this.isStreaming) return;
      const query = this.userInput;
      this.messages.push({ role: 'user', content: query });
      this.userInput = ''; this.isStreaming = true;
      const aiMsg = { role: 'ai', content: '', timeline: [], hasThought: false, isDoneThinking: false, isThoughtExpanded: true, thinkingDuration: 0.0, startTime: Date.now() };
      this.messages.push(aiMsg);
      this.scrollToBottom();

      this.timerInterval = setInterval(() => {
        if (!aiMsg.isDoneThinking) aiMsg.thinkingDuration = ((Date.now() - aiMsg.startTime) / 1000).toFixed(1);
      }, 100);

      try {
        const res = await fetch('http://localhost:8000/chat/stream', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ query, thread_id: this.threadId })
        });
        const reader = res.body.getReader();
        const decoder = new TextDecoder();
        let curBlock = null;

        /* eslint-disable-next-line no-constant-condition */
        while (true) {
          const { done, value } = await reader.read();
          if (done) break;
          const chunks = decoder.decode(value).split('\n\n');
          for (const chunk of chunks) {
            if (!chunk.trim()) continue;
            const eventPart = chunk.split('\n')[0];
            const event = eventPart ? eventPart.replace('event: ', '') : '';
            const dataPart = chunk.split('\ndata: ')[1];
            const data = JSON.parse(dataPart || '{}');

            if (event === 'step') {
              aiMsg.hasThought = true;
              curBlock = { title: data.title, status: data.status || 'loading', content: '' };
              aiMsg.timeline.push(curBlock);
              this.scrollToBottom();
            } else if (event === 'thought') {
              aiMsg.hasThought = true;
              if (!curBlock) {
                curBlock = { title: '思考中', status: 'loading', content: '' };
                aiMsg.timeline.push(curBlock);
              }
              let text = data.content;
              if (curBlock.content === '' && text.trim().startsWith('Content:')) text = text.replace(/^\s*Content:\s*/, '');
              curBlock.content += text;
              this.scrollToBottom();
            } else if (event === 'message') {
              if (!aiMsg.isDoneThinking) {
                aiMsg.isDoneThinking = true;
                clearInterval(this.timerInterval);
                setTimeout(() => { aiMsg.isThoughtExpanded = false; }, 1500);
              }
              aiMsg.content += data.content;
              this.scrollToBottom();
            }
          }
        }
      } catch (err) {
        console.error('Stream error:', err);
      } finally {
        this.isStreaming = false; aiMsg.isDoneThinking = true; clearInterval(this.timerInterval);
      }
    },
    scrollToBottom() { this.$nextTick(() => { const b = this.$refs.chatBox; if (b) b.scrollTo({ top: b.scrollHeight, behavior: 'smooth' }); }); }
  }
};
</script>

<style scoped>
.chat-container { display: flex; flex-direction: column; height: 100%; width: 100%; position: relative; background: #f8f8f7; }
.chat-header { height: 56px; }
.chat-box { flex: 1; overflow-y: auto; padding: 20px 40px; padding-bottom: 240px; display: flex; flex-direction: column; align-items: center; }
.messages-wrapper { width: 85%; max-width: 800px; display: flex; flex-direction: column; }
.input-section { position: absolute; bottom: 40px; left: 50%; transform: translateX(-50%); width: 85%; max-width: 800px; }
.custom-input-box { background: #fff; border-radius: 20px; box-shadow: 0 4px 20px rgba(0,0,0,0.08); padding: 15px 20px; display: flex; align-items: flex-end; }
textarea { flex: 1; border: none; outline: none; resize: none; height: 60px; font-size: 16px; }
.play-icon { width: 28px; cursor: pointer; margin-left: 10px; }
.message { margin-bottom: 25px; width: 100%; display: flex; flex-direction: column; }
.message.user { align-items: flex-end; }
.user .message-content { background: #eef2f6; padding: 12px 18px; border-radius: 18px; max-width: 80%; }
.ai .message-content { background: #fff; padding: 15px 20px; border-radius: 18px; border-top-left-radius: 2px; border: 1px solid #f0f0f0; width: 100%; box-sizing: border-box; }
.ai-label { font-size: 12px; color: #999; margin-bottom: 5px; font-weight: bold; }

/* 思考过程 */
.thought-process { background: #fcfdfe; border: 1px solid #eef2f6; border-radius: 12px; margin: 10px 0; }
.thought-header { padding: 10px 15px; display: flex; align-items: center; cursor: pointer; border-bottom: 1px solid #f1f5f9; }
.thinking-spinner-modern { width: 14px; height: 14px; border: 2px solid #e2e8f0; border-top-color: #3b82f6; border-radius: 50%; animation: spin 0.8s linear infinite; margin-right: 10px; }
.status-title { font-size: 13px; font-weight: 600; color: #475569; }
.status-timer { margin-left: 8px; font-size: 11px; color: #94a3b8; }
.timeline-container { position: relative; padding: 15px 10px 15px 25px; }
.timeline-container::before { content: ''; position: absolute; left: 16px; top: 20px; bottom: 20px; width: 1px; background: #e2e8f0; }
.thought-node { position: relative; margin-bottom: 15px; }
.node-header { display: flex; align-items: center; margin-bottom: 5px; }
.node-icon { position: absolute; left: -14px; background: #fff; padding: 2px 0; }
.spinner-small { width: 10px; height: 10px; border: 2px solid #e2e8f0; border-top-color: #3b82f6; border-radius: 50%; animation: spin 1s linear infinite; display: block; }
.icon-success { color: #10b981; font-size: 12px; font-weight: bold; }
.node-title { font-size: 13px; font-weight: 600; color: #334155; }
.node-body { font-size: 13px; color: #64748b; line-height: 1.6; }

@keyframes spin { to { transform: rotate(360deg); } }
.toggle-arrow { margin-left: auto; transition: transform 0.3s; }
.toggle-arrow.is-expanded { transform: rotate(180deg); }
.is-streaming::after { content: '▋'; animation: blink 1s step-end infinite; color: #3b82f6; }
@keyframes blink { 50% { opacity: 0; } }
</style>