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

  <!-- 聊天界面 -->
  <div v-else class="chat-container" @mousemove="resetTimer" @click="resetTimer" @keydown="resetTimer">
    <div class="chat-header-custom">
      <span class="app-title">FJAD审图</span>
      <span class="chat-with">我说了算</span>
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
      <input
        type="text"
        v-model="inputText"
        placeholder="输入消息..."
        @keyup.enter="sendTextMessage"
      />
      <label class="file-btn">
        📎
        <input type="file" accept="image/*" @change="sendImageMessage" style="display: none;" />
      </label>
      <button @click="sendTextMessage">发送</button>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, nextTick } from 'vue';
import TencentCloudChat from '@tencentcloud/chat';

// ========== 请替换为你的腾讯云 IM 配置 ==========
const SDKAppID = 1600133534;               // 替换为你的 SDKAppID
const myUserID = 'gongjuren';     
const userSig = 'eJwtzDsPgjAYheH-0tlg6UVSEgeJkkAMi7i5NLaWT*USaNFg-O8iMJ7nJO8H5ceT1*sWhYh4GK2mDUpXFm4wsa3rZymr5erUQzYNKBT6G4x9Sjll86PfDbR6dM45wRjPaqH8WxAwJpgQZKmAGcvOuNRlXQvDZX1Psus*zXdysNTFsTaRiF59YY0YknNRHrbo*wMTYTPk';            // 替换为你的 UserSig
const targetUserID = 'woshuolesuan';
// =============================================

let chat = null;
const showChat = ref(false);
const password = ref('');
const errorMsg = ref('');
const inputText = ref('');
const messages = ref([]);
const messageListRef = ref(null);

// 防偷窥计时
let inactivityTimer = null;
const INACTIVE_LIMIT = 2 * 60 * 1000;

// 通知权限
let notificationPermission = false;

const resetTimer = () => {
  if (!showChat.value) return;
  if (inactivityTimer) clearTimeout(inactivityTimer);
  inactivityTimer = setTimeout(() => {
    showChat.value = false;
    if (chat) chat.logout();
  }, INACTIVE_LIMIT);
};

// 格式化时间
const formatTime = (timestamp) => {
  const date = new Date(timestamp * 1000);
  return `${date.getHours().toString().padStart(2,'0')}:${date.getMinutes().toString().padStart(2,'0')}`;
};

// 添加消息到列表
const addMessage = (message) => {
  messages.value.push(message);
  nextTick(() => {
    if (messageListRef.value) {
      messageListRef.value.scrollTop = messageListRef.value.scrollHeight;
    }
  });
};

// 发送文本消息
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

// 发送图片消息
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
  event.target.value = ''; // 清空 input，允许重复上传同一文件
};

// 上报已读（收到消息后调用）
const reportMessageRead = (messageList) => {
  if (messageList.length === 0) return;
  const lastMsg = messageList[messageList.length - 1];
  if (lastMsg.flow === 'in') {
    chat.setMessageRead({ conversationID: `C2C${targetUserID}` });
  }
};

// 监听新消息
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

// 监听消息已读回执（更新已读状态）
const onMessageReadReceipt = (event) => {
  const receipts = event.data;
  receipts.forEach(receipt => {
    const msg = messages.value.find(m => m.ID === receipt.messageID);
    if (msg) msg.isPeerRead = receipt.isPeerRead;
  });
};

// 登录 IM
const loginIM = async () => {
  chat = TencentCloudChat.create({ SDKAppID });
  chat.setLogLevel(0); // 减少日志输出
  await chat.login({ userID: myUserID, userSig });
  // 获取历史消息
  const res = await chat.getMessageList({ conversationID: `C2C${targetUserID}`, count: 20 });
  messages.value = res.data.messageList.reverse(); // 正序显示
  reportMessageRead(messages.value);
  // 注册事件
  chat.on(TencentCloudChat.EVENT.MESSAGE_RECEIVED, onMessageReceived);
  chat.on(TencentCloudChat.EVENT.MESSAGE_READ_RECEIPT, onMessageReadReceipt);
  resetTimer();
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
    loginIM();
    return;
  }
  errorMsg.value = '密码错误，无法进入';
};

// 请求通知权限
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
/* 密码界面样式（与之前一致） */
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
.app-title { font-size: 1.2rem; color: #e9ecef; }
.chat-with { color: #b9bbbe; }

/* 消息列表 */
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
  padding: 12px;
  display: flex;
  gap: 8px;
  align-items: center;
  border-top: 1px solid #3a3f44;
}
.input-area input {
  flex: 1;
  background: #3a3f44;
  border: none;
  padding: 10px;
  border-radius: 20px;
  color: white;
}
.input-area button, .file-btn {
  background: #5e5ce0;
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
.file-btn {
  background: #4a4e54;
  text-decoration: none;
  font-size: 18px;
}
</style>
