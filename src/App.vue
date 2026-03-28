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
    <div class="chat-header-custom">
      <span class="app-title">FJAD审图</span>
      <span class="chat-with">与 {{ targetUserNick }} 聊天</span>
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
        <label class="file-btn">
          📎
          <input type="file" accept="image/*" @change="sendImageMessage" style="display: none;" />
        </label>
        <button @click="sendTextMessage">发送</button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, nextTick } from 'vue';
import TencentCloudChat from '@tencentcloud/chat';

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
const INACTIVE_LIMIT = 2 * 60 * 1000;
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
  showChat.value = false;
  showConfig.value = true;
  isLoggedIn.value = false;
  if (inactivityTimer) clearTimeout(inactivityTimer);
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
    chat.setLogLevel(0);
    await chat.login({ userID, userSig });
    // 等待 SDK 就绪
    await new Promise((resolve) => {
      if (chat.isReady()) resolve();
      else chat.on(TencentCloudChat.EVENT.SDK_READY, resolve);
    });
    console.log('SDK ready');

    const conversationID = `C2C${targetID}`;
    const res = await chat.getMessageList({ conversationID, count: 20 });
    // 消息按时间升序（旧在上，新在下）
    messages.value = res.data.messageList.slice().sort((a, b) => a.time - b.time);
    reportMessageRead(messages.value);

    // 注册事件（使用官方常量，确保存在）
    chat.on(TencentCloudChat.EVENT.MESSAGE_RECEIVED, onMessageReceived);
    chat.on(TencentCloudChat.EVENT.MESSAGE_READ_RECEIPT, onMessageReadReceipt);
    console.log('事件已注册');

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
    showChat.value = false;
    isLoggedIn.value = false;
  }, INACTIVE_LIMIT);
};

const formatTime = (timestamp) => {
  const date = new Date(timestamp * 1000);
  return `${date.getHours().toString().padStart(2,'0')}:${date.getMinutes().toString().padStart(2,'0')}`;
};

// 统一添加消息（自动排序并滚动）
const addMessage = (message) => {
  // 防止重复
  if (messages.value.some(m => m.ID === message.ID)) {
    console.warn('消息已存在，跳过添加', message.ID);
    return;
  }
  messages.value.push(message);
  // 按时间升序排序（时间戳小的在上）
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

const sendImageMessage = async (event) => {
  const file = event.target.files[0];
  if (!file) return;
  const message = chat.createImageMessage({
    to: config.value.targetUserID,
    conversationType: TencentCloudChat.TYPES.CONV_C2C,
    payload: { file }
  });
  try {
    const res = await chat.sendMessage(message);
    addMessage(res.data.message);
  } catch (err) {
    console.error('发送图片失败', err);
  }
  event.target.value = '';
};

// 收到新消息
const onMessageReceived = (event) => {
  console.log('🔥 收到新消息事件', event.data);
  const newMessages = event.data;
  if (!newMessages || newMessages.length === 0) return;
  newMessages.forEach(msg => {
    addMessage(msg);
  });
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
  if (chat) chat.logout();
});
</script>

<style scoped>
/* 样式保持不变，从上一版复制完整样式 */
/* 此处省略以节省篇幅，请从上一个完整代码中复制 <style scoped> 部分 */
</style>
