<template>
  <!-- 密码输入界面 -->
  <div v-if="!showChat" class="password-container">
    <div class="password-box">
      <h2>FJAD审图</h2>
      <input type="password" v-model="password" placeholder="请输入密码" @keyup.enter="checkPassword" autofocus />
      <button @click="checkPassword">验证</button>
      <p v-if="errorMsg" class="error">{{ errorMsg }}</p>
    </div>
  </div>

  <!-- 聊天界面 -->
  <div v-else class="chat-container" @mousemove="resetTimer" @click="resetTimer" @keydown="resetTimer">
    <div class="chat-header-custom">
      <span class="app-title">FJAD审图</span>
      <span class="chat-with">我说了算</span>
    </div>
    <Chat
      :SDKAppID="SDKAppID"
      :userID="myUserID"
      :userSig="myUserSig"
      @login-success="onLoginSuccess"
    >
      <MessageList />
      <MessageInput />
    </Chat>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue';
import { Chat, MessageList, MessageInput } from './TUIKit';
import './TUIKit/styles/index.css';

// ========== ⚠️ 请替换为你的腾讯云IM配置 ==========
const SDKAppID = 1600133534;               // 替换为你的 SDKAppID
const myUserID = 'toolman';     
const myUserSig = 'eJwtzDsPgjAYheH-0tlg6UVSEgeJkkAMi7i5NLaWT*USaNFg-O8iMJ7nJO8H5ceT1*sWhYh4GK2mDUpXFm4wsa3rZymr5erUQzYNKBT6G4x9Sjll86PfDbR6dM45wRjPaqH8WxAwJpgQZKmAGcvOuNRlXQvDZX1Psus*zXdysNTFsTaRiF59YY0YknNRHrbo*wMTYTPk';          // 替换为你的 UserSig
const targetUserID = 'queen';
// =============================================

const showChat = ref(false);
const password = ref('');
const errorMsg = ref('');
let inactivityTimer = null;
const INACTIVE_LIMIT = 2 * 60 * 1000;
let notificationPermission = false;
let chatInstance = null;

const resetTimer = () => {
  if (!showChat.value) return;
  if (inactivityTimer) clearTimeout(inactivityTimer);
  inactivityTimer = setTimeout(() => {
    showChat.value = false;
    chatInstance = null;
  }, INACTIVE_LIMIT);
};

const requestNotificationPermission = () => {
  if ('Notification' in window) {
    Notification.requestPermission().then(perm => {
      notificationPermission = perm === 'granted';
    });
  }
};

const showNotification = (title, body) => {
  if (notificationPermission && document.hidden) {
    new Notification(title, { body });
  }
};

const setupMessageListener = (chat) => {
  chat.on(chat.EVENT.MESSAGE_RECEIVED, (event) => {
    const messages = event.data;
    if (messages && messages.length) {
      showNotification('新消息', messages[0].payload?.text || '您收到一条新消息');
    }
  });
};

const onLoginSuccess = (event) => {
  chatInstance = event.chat;
  chatInstance.updateMyProfile({ nick: '工具人', avatar: '' });
  const conversationID = `C2C${targetUserID}`;
  const conversation = chatInstance.getConversation(conversationID);
  if (conversation) {
    conversation.showName = '我说了算';
  } else {
    chatInstance.createConversation({ type: 'C2C', userID: targetUserID }).then(conv => {
      conv.showName = '我说了算';
      chatInstance.setActiveConversation(conv.conversationID);
    });
  }
  chatInstance.setActiveConversation(conversationID);
  setupMessageListener(chatInstance);
  resetTimer();
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

onMounted(() => {
  requestNotificationPermission();
});

onUnmounted(() => {
  if (inactivityTimer) clearTimeout(inactivityTimer);
});
</script>

<style scoped>
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
.password-box h2 { color: #e9ecef; margin-bottom: 1.5rem; }
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
.error { color: #e74c3c; margin-top: 1rem; }
.chat-container { height: 100vh; display: flex; flex-direction: column; background: #2c2f33; }
.chat-header-custom {
  background: #23272a;
  padding: 12px 16px;
  display: flex;
  justify-content: space-between;
  border-bottom: 1px solid #3a3f44;
}
.app-title { font-size: 1.2rem; color: #e9ecef; }
.chat-with { color: #b9bbbe; }
:deep(.chat-header) { display: none !important; }
:deep(.avatar) { display: none !important; }
</style>
