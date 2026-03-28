<template>
  <!-- 密码界面 -->
  <div v-if="!showChat" class="password-container">
    <div class="password-box">
      <h2>FJAD审图</h2>
      <input
        type="password"
        v-model="password"
        placeholder="请输入密码"
        @keyup.enter="checkPassword"
        autofocus
      />
      <button @click="checkPassword">验证</button>
      <p v-if="errorMsg" class="error">{{ errorMsg }}</p>
    </div>
  </div>

  <!-- 配置界面 -->
  <div v-else-if="showConfig" class="config-container">
    <div class="config-box">
      <h2>聊天设置</h2>
      <div class="config-field">
        <label>SDKAppID</label>
        <input v-model="config.SDKAppID" type="number" placeholder="例如 1400123456" />
        <small>在腾讯云 IM 控制台 → 应用信息 中查看</small>
      </div>
      <div class="config-field">
        <label>自己的 userID</label>
        <input v-model="config.myUserID" placeholder="例如 gongjuren" />
        <small>只能包含字母、数字、下划线</small>
      </div>
      <div class="config-field">
        <label>对方的 userID</label>
        <input v-model="config.targetUserID" placeholder="例如 woshuolesuan" />
      </div>
      <div class="config-field">
        <label>自己的 userSig</label>
        <textarea v-model="config.myUserSigRaw" rows="2" placeholder="从控制台复制，很长一串"></textarea>
        <small><a href="https://console.cloud.tencent.com/im/tool-usersig" target="_blank">点击获取 UserSig</a>（需登录腾讯云）</small>
        <small>注意：复制后不要带引号、换行符等额外字符</small>
      </div>
      <div class="config-buttons">
        <button @click="saveAndStart" :disabled="loadingLogin">保存并开始聊天</button>
        <button v-if="hasSavedConfig" @click="resetConfig">重置配置</button>
      </div>
      <p v-if="configError" class="error">{{ configError }}</p>
    </div>
  </div>

  <!-- 聊天界面 -->
  <div v-else class="chat-container" @mousemove="resetTimer" @click="resetTimer" @keydown="resetTimer">
    <!-- 头部只保留居中设置按钮 -->
    <div class="chat-header-custom">
      <button class="settings-btn" @click="openSettings">⚙️</button>
    </div>

    <div class="message-list" ref="messageListRef">
      <div v-for="msg in messages" :key="msg.ID" :class="['message', msg.flow === 'out' ? 'sent' : 'received']">
        <div class="bubble">
          <div v-if="msg.type === 'TIMTextElem'">{{ msg.payload.text }}</div>
          <div v-else-if="msg.type === 'TIMImageElem'">
            <img :src="msg.payload.imageInfoArray[0].url" style="max-width: 200px; max-height: 200px;" />
          </div>
          <div v-else>不支持的消息类型</div>
          <div class="time">{{ formatTime(msg.time) }}</div>
          <div v-if="msg.flow === 'out'" class="read-status">
            {{ msg.isPeerRead ? '已读' : '未读' }}
          </div>
        </div>
      </div>
    </div>

    <div class="input-area">
      <div class="emoji-panel" v-if="showEmoji">
        <button v-for="emoji in emojiList" :key="emoji" @click="insertEmoji(emoji)">{{ emoji }}</button>
      </div>
      <div class="input-row">
        <input
          type="text"
          v-model="inputText"
          placeholder="输入消息..."
          @keyup.enter="sendTextMessage"
          ref="messageInput"
        />
        <button class="emoji-btn" @click="toggleEmojiPanel">😀</button>
        <button @click="sendTextMessage">发送</button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, nextTick } from 'vue';
import TencentCloudChat from '@tencentcloud/chat';
import COS from 'cos-js-sdk-v5';

const defaultConfig = {
  SDKAppID: 0,
  myUserID: '',
  targetUserID: '',
  myUserSigRaw: ''
};

// 页面状态
const showChat = ref(false);
const showConfig = ref(false);
const isLoggedIn = ref(false);
const loadingLogin = ref(false);
const configError = ref('');
const password = ref('');
const errorMsg = ref('');

// 聊天相关
const inputText = ref('');
const messages = ref([]);
const messageListRef = ref(null);
const messageInput = ref(null);
let chat = null;
let inactivityTimer = null;
let pollTimer = null;
const INACTIVE_LIMIT = 2 * 60 * 1000;
const POLL_INTERVAL = 3000;
let notificationPermission = false;

const config = ref({ ...defaultConfig });
const hasSavedConfig = ref(false);
const targetUserNick = ref('');

// 表情列表
const emojiList = [
  '😀', '😃', '😄', '😁', '😆', '😅', '😂', '🤣', '😊', '😇',
  '🙂', '🙃', '😉', '😌', '😍', '🥰', '😘', '😗', '😙', '😚',
  '😋', '😛', '😝', '😜', '🤪', '🤨', '🧐', '🤓', '😎', '🤩',
  '🥳', '😏', '😒', '😞', '😔', '😟', '😕', '🙁', '☹️', '😣',
  '😖', '😫', '😩', '🥺', '😢', '😭', '😤', '😠', '😡', '🤬',
  '🤯', '😳', '🥵', '🥶', '😱', '😨', '😰', '😥', '😓', '🤗',
  '🤔', '🤭', '🤫', '🤥', '😶', '😐', '😑', '😬', '🙄', '😯',
  '😦', '😧', '😮', '😲', '🥱', '😴', '🤤', '😪', '😵', '🤐',
  '🥴', '🤢', '🤮', '🤧', '😷', '🤒', '🤕', '🤑', '🤠', '😈',
  '👿', '👹', '👺', '💀', '👻', '👽', '🤖', '🎃', '😺', '😸'
];
const showEmoji = ref(false);
const toggleEmojiPanel = () => { showEmoji.value = !showEmoji.value; };
const insertEmoji = (emoji) => {
  inputText.value += emoji;
  showEmoji.value = false;
  nextTick(() => messageInput.value.focus());
};

// 清理 userSig
const cleanUserSig = (raw) => {
  if (!raw) return '';
  let cleaned = raw.trim();
  if ((cleaned.startsWith('"') && cleaned.endsWith('"')) || (cleaned.startsWith("'") && cleaned.endsWith("'"))) {
    cleaned = cleaned.slice(1, -1);
  }
  cleaned = cleaned.replace(/[\r\n]/g, '');
  return cleaned;
};

// 加载本地配置
const loadConfig = () => {
  const saved = localStorage.getItem('fjad_chat_config');
  if (saved) {
    try {
      const parsed = JSON.parse(saved);
      config.value = {
        ...defaultConfig,
        SDKAppID: parsed.SDKAppID,
        myUserID: (parsed.myUserID || '').trim(),
        targetUserID: (parsed.targetUserID || '').trim(),
        myUserSigRaw: parsed.myUserSigRaw || ''
      };
      hasSavedConfig.value = true;
      targetUserNick.value = config.value.targetUserID;
      return true;
    } catch (e) {
      console.warn('配置读取失败', e);
    }
  }
  return false;
};

// 保存配置并开始聊天
const saveAndStart = async () => {
  configError.value = '';
  let sdkAppIdStr = (config.value.SDKAppID || '').toString().trim();
  let myUserID = (config.value.myUserID || '').trim();
  let targetUserID = (config.value.targetUserID || '').trim();
  let myUserSigRaw = config.value.myUserSigRaw || '';

  let myUserSig = cleanUserSig(myUserSigRaw);
  console.log('清理后 userSig 长度:', myUserSig.length);

  if (!sdkAppIdStr) {
    configError.value = '请填写 SDKAppID';
    return;
  }
  const sdkAppIdNum = Number(sdkAppIdStr);
  if (isNaN(sdkAppIdNum)) {
    configError.value = 'SDKAppID 必须是数字';
    return;
  }
  if (!myUserID) {
    configError.value = '请填写自己的 userID';
    return;
  }
  if (!targetUserID) {
    configError.value = '请填写对方的 userID';
    return;
  }
  if (!myUserSig) {
    configError.value = '请填写自己的 userSig';
    return;
  }

  config.value = {
    SDKAppID: sdkAppIdNum,
    myUserID,
    targetUserID,
    myUserSigRaw: myUserSigRaw
  };
  localStorage.setItem('fjad_chat_config', JSON.stringify(config.value));
  hasSavedConfig.value = true;
  targetUserNick.value = targetUserID;
  showConfig.value = false;
  await startChat();
};

// 重置配置
const resetConfig = () => {
  localStorage.removeItem('fjad_chat_config');
  config.value = { ...defaultConfig };
  hasSavedConfig.value = false;
  showConfig.value = true;
};

// 打开设置
const openSettings = () => {
  if (chat) {
    chat.logout();
    chat = null;
  }
  if (pollTimer) clearInterval(pollTimer);
  showChat.value = false;
  showConfig.value = true;
  isLoggedIn.value = false;
  if (inactivityTimer) clearTimeout(inactivityTimer);
};

// 拉取新消息（轮询用）
const fetchNewMessages = async () => {
  if (!chat || !isLoggedIn.value) return;
  try {
    const conversationID = `C2C${config.value.targetUserID}`;
    const lastMsg = messages.value[messages.value.length - 1];
    const lastMsgTime = lastMsg ? lastMsg.time : 0;
    const res = await chat.getMessageList({ conversationID, count: 20, lastMsgTime });
    if (res.data.messageList && res.data.messageList.length > 0) {
      const existingIds = new Set(messages.value.map(m => m.ID));
      const newMsgs = res.data.messageList.filter(msg => !existingIds.has(msg.ID));
      if (newMsgs.length > 0) {
        newMsgs.forEach(msg => addMessage(msg));
        reportMessageRead(newMsgs);
        if (notificationPermission && document.hidden) {
          new Notification('新消息', { body: newMsgs[0]?.payload?.text || '您收到一条新消息' });
        }
      }
    }
  } catch (err) {
    console.error('轮询拉取消息失败', err);
  }
};

// 登录 IM 并开始聊天
const startChat = async () => {
  if (loadingLogin.value) return;
  loadingLogin.value = true;
  configError.value = '';
  try {
    const sdkAppId = config.value.SDKAppID;
    const userID = config.value.myUserID;
    const userSig = cleanUserSig(config.value.myUserSigRaw);
    const targetID = config.value.targetUserID;

    console.log('准备登录，userSig长度:', userSig.length);
    if (!userSig) throw new Error('userSig 不能为空');

    if (chat) {
      await chat.logout();
      chat = null;
    }
    chat = TencentCloudChat.create({ SDKAppID: sdkAppId });
    
    // 注册 COS 插件（图片可选，不影响文字）
    try {
      const cos = new COS({
        getAuthorization: (options, callback) => {
          callback({ Authorization: '', SecurityToken: '' });
        }
      });
      chat.registerPlugin({ 'cos': cos });
    } catch (err) {
      console.warn('COS 插件注册失败，图片可能无法发送', err);
    }
    
    chat.setLogLevel(0);
    await chat.login({ userID, userSig });
    await new Promise((resolve) => {
      if (chat.isReady()) resolve();
      else chat.on(TencentCloudChat.EVENT.SDK_READY, resolve);
    });
    console.log('SDK ready');

    const conversationID = `C2C${targetID}`;
    const res = await chat.getMessageList({ conversationID, count: 20 });
    messages.value = res.data.messageList.slice().sort((a, b) => a.time - b.time);
    reportMessageRead(messages.value);

    // 注册事件
    chat.on('MESSAGE_RECEIVED', onMessageReceived);
    chat.on('MESSAGE_READ_RECEIPT', onMessageReadReceipt);
    console.log('事件已注册');

    // 启动轮询
    if (pollTimer) clearInterval(pollTimer);
    pollTimer = setInterval(fetchNewMessages, POLL_INTERVAL);
    console.log(`轮询已启动，间隔 ${POLL_INTERVAL}ms`);

    isLoggedIn.value = true;
    resetTimer();
  } catch (err) {
    console.error('登录失败', err);
    let errorDetail = err.message || err.code || '参数验证失败，请检查 SDKAppID、userID 和 userSig';
    configError.value = `登录失败：${errorDetail}`;
    showConfig.value = true;
    if (chat) {
      await chat.logout().catch(() => {});
      chat = null;
    }
  } finally {
    loadingLogin.value = false;
  }
};

// 防偷窥计时
const resetTimer = () => {
  if (!isLoggedIn.value) return;
  if (inactivityTimer) clearTimeout(inactivityTimer);
  inactivityTimer = setTimeout(() => {
    if (chat) chat.logout();
    if (pollTimer) clearInterval(pollTimer);
    showChat.value = false;
    isLoggedIn.value = false;
  }, INACTIVE_LIMIT);
};

const formatTime = (timestamp) => {
  const date = new Date(timestamp * 1000);
  return `${date.getHours().toString().padStart(2,'0')}:${date.getMinutes().toString().padStart(2,'0')}`;
};

// 统一添加消息
const addMessage = (message) => {
  if (messages.value.some(m => m.ID === message.ID)) return;
  messages.value.push(message);
  messages.value.sort((a, b) => a.time - b.time);
  nextTick(() => {
    if (messageListRef.value) {
      messageListRef.value.scrollTop = messageListRef.value.scrollHeight;
    }
  });
};

const sendTextMessage = async () => {
  const text = inputText.value.trim();
  if (!text) return;
  const message = chat.createTextMessage({
    to: config.value.targetUserID,
    conversationType: TencentCloudChat.TYPES.CONV_C2C,
    payload: { text }
  });
  try {
    const res = await chat.sendMessage(message);
    addMessage(res.data.message);
    inputText.value = '';
  } catch (err) {
    console.error('发送失败', err);
  }
};

// 收到新消息
const onMessageReceived = (event) => {
  console.log('🔥 收到新消息事件（SDK触发）', event.data);
  const newMessages = event.data;
  if (!newMessages || newMessages.length === 0) return;
  newMessages.forEach(msg => addMessage(msg));
  reportMessageRead(newMessages);
  if (notificationPermission && document.hidden) {
    new Notification('新消息', { body: newMessages[0]?.payload?.text || '您收到一条新消息' });
  }
};

// 已读回执
const onMessageReadReceipt = (event) => {
  const receipts = event.data;
  receipts.forEach(receipt => {
    const msg = messages.value.find(m => m.ID === receipt.messageID);
    if (msg) msg.isPeerRead = receipt.isPeerRead;
  });
};

const reportMessageRead = (messageList) => {
  if (!messageList.length) return;
  const lastMsg = messageList[messageList.length - 1];
  if (lastMsg.flow === 'in') {
    chat.setMessageRead({ conversationID: `C2C${config.value.targetUserID}` });
  }
};

// 密码验证
const checkPassword = () => {
  const pwd = password.value;
  errorMsg.value = '';
  if (pwd === '19532026') {
    window.location.href = 'http://175.42.20.145:8081';
    return;
  }
  const len = pwd.length;
  if (len >= 4 && pwd[1] === '1' && pwd[2] === '3' && pwd[len-4] === '5' && pwd[len-3] === '7') {
    showChat.value = true;
    password.value = '';
    const hasConfig = loadConfig();
    if (!hasConfig) {
      showConfig.value = true;
    } else {
      startChat();
    }
    return;
  }
  errorMsg.value = '密码错误，无法进入';
};

const requestNotification = () => {
  if ('Notification' in window) {
    Notification.requestPermission().then(perm => notificationPermission = perm === 'granted');
  }
};

onMounted(() => {
  requestNotification();
});

onUnmounted(() => {
  if (inactivityTimer) clearTimeout(inactivityTimer);
  if (pollTimer) clearInterval(pollTimer);
  if (chat) chat.logout();
});
</script>

<style scoped>
/* 密码界面 */
.password-container {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
  background: #2c2f33;
  padding: 20px;
}
.password-box {
  background: #23272a;
  padding: 2rem;
  border-radius: 16px;
  text-align: center;
  width: 100%;
  max-width: 320px;
  box-shadow: 0 4px 12px rgba(0,0,0,0.3);
}
.password-box h2 {
  color: #e9ecef;
  margin-bottom: 1.5rem;
  font-size: 1.5rem;
}
.password-box input {
  width: 100%;
  padding: 14px;
  margin-bottom: 1rem;
  border: 1px solid #4a4e54;
  background: #2c2f33;
  color: #fff;
  border-radius: 30px;
  font-size: 1rem;
  outline: none;
  text-align: center;
}
.password-box button {
  width: 100%;
  padding: 14px;
  background: #4a4e54;
  color: white;
  border: none;
  border-radius: 30px;
  font-size: 1rem;
  cursor: pointer;
  transition: 0.2s;
}
.password-box button:active {
  background: #5a5e64;
  transform: scale(0.98);
}
.error {
  color: #e74c3c;
  margin-top: 1rem;
  font-size: 0.85rem;
}

/* 配置界面 */
.config-container {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
  background: #2c2f33;
  padding: 20px;
  overflow-y: auto;
}
.config-box {
  background: #23272a;
  padding: 1.5rem;
  border-radius: 20px;
  width: 100%;
  max-width: 400px;
  margin: auto;
}
.config-field {
  margin-bottom: 1.2rem;
}
.config-field label {
  display: block;
  color: #e9ecef;
  margin-bottom: 6px;
  font-size: 0.9rem;
  font-weight: 500;
}
.config-field input, .config-field textarea {
  width: 100%;
  padding: 12px;
  background: #2c2f33;
  border: 1px solid #4a4e54;
  color: #fff;
  border-radius: 12px;
  font-size: 0.95rem;
  box-sizing: border-box;
}
.config-field textarea {
  resize: vertical;
  min-height: 80px;
}
.config-field small {
  display: block;
  color: #b9bbbe;
  font-size: 0.7rem;
  margin-top: 4px;
}
.config-buttons {
  display: flex;
  gap: 12px;
  justify-content: center;
  margin-top: 1.5rem;
}
.config-buttons button {
  flex: 1;
  padding: 12px;
  background: #4a4e54;
  color: white;
  border: none;
  border-radius: 30px;
  font-size: 1rem;
  cursor: pointer;
  transition: 0.2s;
}
.config-buttons button:active {
  background: #5a5e64;
  transform: scale(0.97);
}

/* 聊天界面 */
.chat-container {
  height: 100vh;
  display: flex;
  flex-direction: column;
  background: #2c2f33;
  position: relative;
}
.chat-header-custom {
  background: #23272a;
  padding: 14px 16px;
  display: flex;
  justify-content: center;
  align-items: center;
  border-bottom: 1px solid #3a3f44;
  flex-shrink: 0;
}
.settings-btn {
  background: none;
  border: none;
  color: #b9bbbe;
  font-size: 1.4rem;
  cursor: pointer;
  padding: 8px;
}
.settings-btn:active {
  color: #e9ecef;
}

/* 消息列表 */
.message-list {
  flex: 1;
  overflow-y: auto;
  padding: 16px;
  display: flex;
  flex-direction: column;
  gap: 12px;
  -webkit-overflow-scrolling: touch;
}
.message {
  display: flex;
  max-width: 85%;
}
.message.sent {
  align-self: flex-end;
  flex-direction: row-reverse;
}
.message.received {
  align-self: flex-start;
}
.bubble {
  background: #3a3f44;
  color: #fff;
  padding: 10px 14px;
  border-radius: 20px;
  position: relative;
  word-break: break-word;
  max-width: 100%;
}
.sent .bubble {
  background: #5e5ce0;
  border-bottom-right-radius: 4px;
}
.received .bubble {
  border-bottom-left-radius: 4px;
}
.time {
  font-size: 0.65rem;
  color: #bbb;
  margin-top: 4px;
  text-align: right;
}
.read-status {
  font-size: 0.6rem;
  color: #aaa;
  margin-top: 2px;
  text-align: right;
}

/* 输入区 */
.input-area {
  background: #23272a;
  border-top: 1px solid #3a3f44;
  display: flex;
  flex-direction: column;
  flex-shrink: 0;
}
.emoji-panel {
  padding: 8px;
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  border-bottom: 1px solid #3a3f44;
  max-height: 120px;
  overflow-y: auto;
}
.emoji-panel button {
  background: #3a3f44;
  border: none;
  font-size: 1.6rem;
  cursor: pointer;
  padding: 6px;
  border-radius: 12px;
  width: 44px;
  height: 44px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
}
.emoji-panel button:active {
  background: #5e5ce0;
}
.input-row {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 12px;
}
.input-row input {
  flex: 1;
  background: #3a3f44;
  border: none;
  padding: 12px 16px;
  border-radius: 30px;
  color: white;
  font-size: 1rem;
  line-height: 1.4;
}
.input-row input:focus {
  outline: none;
  background: #4a4e54;
}
.input-row button {
  background: #4a4e54;
  border: none;
  color: white;
  width: 44px;
  height: 44px;
  border-radius: 50%;
  cursor: pointer;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  font-size: 1rem;
  transition: 0.1s;
}
.input-row button:active {
  transform: scale(0.95);
  background: #5a5e64;
}
.emoji-btn {
  background: #5e5ce0;
}
</style>
