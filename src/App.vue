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

  <!-- 身份选择界面 -->
  <div v-else-if="!isLoggedIn" class="role-container">
    <div class="role-box">
      <h2>选择你的身份</h2>
      <button @click="selectRole('gongjuren')" :disabled="loadingRole">工具人</button>
      <button @click="selectRole('woshuolesuan')" :disabled="loadingRole">我说了算</button>
      <p v-if="roleError" class="error">{{ roleError }}</p>
    </div>
  </div>

  <!-- 聊天界面 -->
  <div v-else class="chat-container" @mousemove="resetTimer" @click="resetTimer" @keydown="resetTimer">
    <div class="chat-header-custom">
      <span class="app-title">FJAD审图</span>
      <span class="chat-with">{{ oppositeNick }}</span>
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

// ========== 请替换为你的腾讯云 IM 配置 ==========
const SDKAppID = 1600133534;               // 你的 SDKAppID
const targetUserID = 'woshuolesuan'; // 对方的固定ID
// =============================================

// ⚠️ 重要：替换为两个用户的真实 UserSig（从腾讯云控制台获取）
const userSigMap = {
  gongjuren: 'eJwtzE0LgkAUheH-MttCr47XIcFNZREIBWYkuAkc7U5ofhNF-z1Tl*c58H7Y2Q*0XtbMYaYGbDluSmTRUkojZ88iU10ti-lsksetLClhjmEDGJwjt6ZHvkqq5eCIaALApC3lfxPCQrQRVnOFsqG9D7E59ekV1iFfdEfxVrGe*yKozO1mF1w8dagidfdkF*tp5LLvD5lhM4M_',      // 替换为工具人的 UserSig
  woshuolesuan: 'eJwtzLEOgjAUheF36Yoht9CCIXEhuiCyQBTHJi14AQEpCInx3UVgPN*fnA9Jwth8q454xDKB7JaNUtU9Zrjw2OjH0FRKD6LeupalaFuUxKMOALVtbrO1qKnFTs3OObcAYNUen39zXca5w*imGvP5HuR4xfslOAapmKKmSHwwqqyiYczKfZEYJzGk8Iqi8y0-kO8PGjg0QA__'  // 替换为我说了算的 UserSig
};

let currentUserID = '';
let currentUserSig = '';
let oppositeNick = '';

let chat = null;
const showChat = ref(false);
const isLoggedIn = ref(false);
const loadingRole = ref(false);
const roleError = ref('');

const password = ref('');
const errorMsg = ref('');
const inputText = ref('');
const messages = ref([]);
const messageListRef = ref(null);
const messageInput = ref(null);

// 表情相关
const showEmoji = ref(false);
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

const toggleEmojiPanel = () => {
  showEmoji.value = !showEmoji.value;
};

const insertEmoji = (emoji) => {
  inputText.value += emoji;
  showEmoji.value = false;
  nextTick(() => {
    messageInput.value.focus();
  });
};

// 防偷窥计时
let inactivityTimer = null;
const INACTIVE_LIMIT = 2 * 60 * 1000;

let notificationPermission = false;

const resetTimer = () => {
  if (!showChat.value) return;
  if (inactivityTimer) clearTimeout(inactivityTimer);
  inactivityTimer = setTimeout(() => {
    showChat.value = false;
    if (chat) chat.logout();
  }, INACTIVE_LIMIT);
};

const formatTime = (timestamp) => {
  const date = new Date(timestamp * 1000);
  return `${date.getHours().toString().padStart(2,'0')}:${date.getMinutes().toString().padStart(2,'0')}`;
};

const addMessage = (message) => {
  messages.value.push(message);
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
    to: targetUserID,
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
    to: targetUserID,
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

const onMessageReceived = (event) => {
  const newMessages = event.data;
  newMessages.forEach(msg => {
    addMessage(msg);
  });
  reportMessageRead(newMessages);
  if (notificationPermission && document.hidden) {
    new Notification('新消息', { body: newMessages[0]?.payload?.text || '您收到一条新消息' });
  }
};

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
    chat.setMessageRead({ conversationID: `C2C${targetUserID}` });
  }
};

// 修改 loginIM 函数
const loginIM = async () => {
  if (chat) {
    await chat.logout();
    chat = null;
  }
  chat = TencentCloudChat.create({ SDKAppID });
  chat.setLogLevel(0);
  // 登录
  await chat.login({ userID: currentUserID, userSig: currentUserSig });
  
  // 等待 SDK 就绪
  await new Promise((resolve) => {
    if (chat.isReady()) {
      resolve();
    } else {
      chat.on(TencentCloudChat.EVENT.SDK_READY, resolve);
    }
  });
  
  // 现在可以安全调用 getMessageList
  const conversationID = `C2C${targetUserID}`;
  const res = await chat.getMessageList({ conversationID, count: 20 });
  messages.value = res.data.messageList.reverse();
  reportMessageRead(messages.value);
  
  // 注册事件监听
  chat.on(TencentCloudChat.EVENT.MESSAGE_RECEIVED, onMessageReceived);
  chat.on(TencentCloudChat.EVENT.MESSAGE_READ_RECEIPT, onMessageReadReceipt);
};


const selectRole = async (role) => {
  if (loadingRole.value) return;
  loadingRole.value = true;
  roleError.value = '';
  try {
    currentUserID = role;
    currentUserSig = userSigMap[role];
    // 简单检查 UserSig 是否被替换
    if (currentUserSig.includes('你的工具人UserSig') || currentUserSig.includes('你的我说了算UserSig')) {
      throw new Error('请先在代码中填写正确的 UserSig！');
    }
    oppositeNick = role === 'gongjuren' ? '我说了算' : '工具人';
    await loginIM();
    isLoggedIn.value = true;
    resetTimer();
  } catch (err) {
    console.error('登录失败', err);
    roleError.value = err.message || '登录失败，请检查 UserSig 是否正确';
    // 清理可能残留的聊天实例
    if (chat) {
      await chat.logout().catch(() => {});
      chat = null;
    }
  } finally {
    loadingRole.value = false;
  }
};

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
/* 密码界面 */
.password-container {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
  background: #2c2f33;
}
.password-box {
  background: #23272a;
  padding: 2rem;
  border-radius: 12px;
  text-align: center;
  width: 300px;
}
.password-box h2 {
  color: #e9ecef;
  margin-bottom: 1.5rem;
}
.password-box input {
  width: 100%;
  padding: 10px;
  margin-bottom: 1rem;
  border: 1px solid #4a4e54;
  background: #2c2f33;
  color: #fff;
  border-radius: 6px;
}
.password-box button {
  width: 100%;
  padding: 10px;
  background: #4a4e54;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
}
.error {
  color: #e74c3c;
  margin-top: 1rem;
}

/* 身份选择 */
.role-container {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
  background: #2c2f33;
}
.role-box {
  background: #23272a;
  padding: 2rem;
  border-radius: 12px;
  text-align: center;
  width: 300px;
}
.role-box h2 {
  color: #e9ecef;
  margin-bottom: 1.5rem;
}
.role-box button {
  width: 100%;
  padding: 10px;
  margin: 10px 0;
  background: #4a4e54;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
}
.role-box button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}
.role-box button:hover:not(:disabled) {
  background: #5a5e64;
}

/* 聊天界面 */
.chat-container {
  height: 100vh;
  display: flex;
  flex-direction: column;
  background: #2c2f33;
}
.chat-header-custom {
  background: #23272a;
  padding: 12px 16px;
  display: flex;
  justify-content: space-between;
  border-bottom: 1px solid #3a3f44;
}
.app-title {
  font-size: 1.2rem;
  color: #e9ecef;
}
.chat-with {
  color: #b9bbbe;
}
.message-list {
  flex: 1;
  overflow-y: auto;
  padding: 16px;
  display: flex;
  flex-direction: column;
  gap: 12px;
}
.message {
  display: flex;
  max-width: 80%;
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
  padding: 8px 12px;
  border-radius: 18px;
  position: relative;
}
.sent .bubble {
  background: #5e5ce0;
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
}
.emoji-panel {
  padding: 8px;
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  border-bottom: 1px solid #3a3f44;
  max-height: 150px;
  overflow-y: auto;
}
.emoji-panel button {
  background: #3a3f44;
  border: none;
  font-size: 1.5rem;
  cursor: pointer;
  padding: 4px 8px;
  border-radius: 8px;
  transition: 0.1s;
}
.emoji-panel button:hover {
  background: #5e5ce0;
}
.input-row {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 12px;
}
.input-row input {
  flex: 1;
  background: #3a3f44;
  border: none;
  padding: 10px;
  border-radius: 20px;
  color: white;
}
.input-row button, .file-btn {
  background: #4a4e54;
  border: none;
  color: white;
  width: 36px;
  height: 36px;
  border-radius: 50%;
  cursor: pointer;
  display: inline-flex;
  align-items: center;
  justify-content: center;
}
.emoji-btn {
  background: #5e5ce0;
}
.file-btn {
  background: #4a4e54;
}
</style>
