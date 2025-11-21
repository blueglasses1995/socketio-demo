# Lesson 4: ブロードキャスト

## 🎯 学習目標

- ブロードキャストの概念を理解する
- 全クライアントにメッセージを送信する
- 特定のクライアントを除外してブロードキャストする
- ブロードキャストの様々なパターンを使い分ける

---

## 📋 メリット

✅ **効率的な配信**: 1回の呼び出しで複数クライアントに送信
✅ **コード簡潔性**: ループ不要で複数クライアントに配信
✅ **リアルタイム同期**: 全ユーザーに同時に情報を共有
✅ **柔軟な配信**: 特定のクライアントを除外可能

---

## ⚠️ デメリット

❌ **スケーラビリティ**: クライアント数に比例して負荷増加
❌ **帯域幅消費**: 同じデータを複数回送信
❌ **制御の難しさ**: 個別のエラーハンドリングが困難
❌ **メモリ使用量**: 大量のクライアントで問題になる可能性

---

## ⚙️ 技術的原理

### ブロードキャストの仕組み

```
サーバー
    │
    │ io.emit('event', data)
    │
    ├─────────────┬─────────────┬─────────────┐
    │             │             │             │
    ▼             ▼             ▼             ▼
クライアントA  クライアントB  クライアントC  クライアントD

すべてのクライアントにイベントを送信
```

### 送信元除外ブロードキャスト

```
サーバー
    │
    │ socket.broadcast.emit('event', data)
    │
    │ (socket = クライアントB)
    │
    ├─────────────┬─────────────┬─────────────┐
    │             │             │             │
    ▼             X             ▼             ▼
クライアントA  クライアントB  クライアントC  クライアントD
                (除外)

送信元のクライアント以外に送信
```

### 内部動作

```javascript
// io.emit() の内部処理
1. 接続中の全ソケットのリストを取得
2. 各ソケットに対してループ処理
3. socket.emit() を個別に実行
4. エラーは個別に処理（他のクライアントに影響なし）

// socket.broadcast.emit() の内部処理
1. 接続中の全ソケットのリストを取得
2. 現在のソケットをフィルタで除外
3. 残りのソケットに対してループ処理
4. socket.emit() を個別に実行
```

### メモリとパフォーマンス

- **データのコピー**: 各クライアントに送信する際、データはシリアライズされる（1回のみ）
- **ネットワーク帯域**: クライアント数 × データサイズ の帯域を消費
- **CPU使用**: ソケット数に比例してCPU使用率が上がる

---

## 💼 ユースケース

### 適している場面

1. **チャットアプリ**: 全員にメッセージを配信
2. **通知システム**: 全ユーザーに通知を送信
3. **ライブダッシュボード**: 全クライアントにデータ更新を配信
4. **オンライン状態**: ユーザーのログイン/ログアウトを全員に通知

### 注意が必要な場面

1. **大規模システム**: 数万以上の同時接続（Redis Adapterが必要）
2. **高頻度更新**: 秒間数百回以上の更新（スロットリング推奨）
3. **大容量データ**: MB単位のデータ配信（HTTPの方が適切）
4. **個人情報**: プライバシーに配慮が必要なデータ

---

## 💻 実装例

### ステップ1: プロジェクトのセットアップ

```bash
mkdir socket-broadcast
cd socket-broadcast
npm init -y
npm install express socket.io
```

### ステップ2: サーバー側の実装

`server.js`:

```javascript
const express = require('express');
const { createServer } = require('http');
const { Server } = require('socket.io');

const app = express();
const httpServer = createServer(app);
const io = new Server(httpServer);

app.use(express.static('public'));

// 接続中のユーザー情報を保存
const users = new Map();

io.on('connection', (socket) => {
  console.log(`✅ 接続: ${socket.id}`);

  // ユーザー情報を保存
  users.set(socket.id, {
    id: socket.id,
    username: `ユーザー${users.size + 1}`,
    connectedAt: new Date()
  });

  // ===== パターン1: 全クライアントにブロードキャスト =====

  // 新しいユーザーが参加したことを「全員」に通知
  io.emit('userJoined', {
    user: users.get(socket.id),
    totalUsers: users.size,
    timestamp: new Date().toISOString()
  });

  // ===== パターン2: 送信元を除く全クライアントにブロードキャスト =====

  socket.on('chatMessage', (data) => {
    console.log(`💬 チャットメッセージ: ${data.username}: ${data.message}`);

    // 送信者以外の全クライアントに配信
    socket.broadcast.emit('chatMessage', {
      username: data.username,
      message: data.message,
      timestamp: new Date().toISOString()
    });
  });

  // ===== パターン3: タイピングインジケーター =====

  socket.on('typing', (data) => {
    // 送信者以外に「○○が入力中...」を通知
    socket.broadcast.emit('userTyping', {
      username: data.username,
      isTyping: true
    });
  });

  socket.on('stopTyping', (data) => {
    socket.broadcast.emit('userTyping', {
      username: data.username,
      isTyping: false
    });
  });

  // ===== パターン4: システムアナウンス =====

  socket.on('announcement', (data) => {
    // 管理者からのアナウンスを全員に配信
    io.emit('systemAnnouncement', {
      type: 'info',
      message: data.message,
      from: 'システム',
      timestamp: new Date().toISOString()
    });
  });

  // ===== パターン5: オンラインユーザーリスト =====

  socket.on('requestUserList', () => {
    // リクエスト元のクライアントにのみユーザーリストを送信
    const userList = Array.from(users.values()).map(u => ({
      id: u.id,
      username: u.username
    }));

    socket.emit('userList', userList);
  });

  // ===== パターン6: リアクション（いいね等） =====

  socket.on('reaction', (data) => {
    // 全員にリアクションを配信
    io.emit('reactionReceived', {
      userId: socket.id,
      username: users.get(socket.id).username,
      reactionType: data.type,
      targetMessageId: data.messageId,
      timestamp: Date.now()
    });
  });

  // ===== 切断処理 =====

  socket.on('disconnect', () => {
    console.log(`❌ 切断: ${socket.id}`);

    const user = users.get(socket.id);
    users.delete(socket.id);

    // ユーザーが退出したことを残りの全員に通知
    io.emit('userLeft', {
      user: user,
      totalUsers: users.size,
      timestamp: new Date().toISOString()
    });
  });
});

const PORT = 3000;
httpServer.listen(PORT, () => {
  console.log(`🚀 サーバー起動: http://localhost:${PORT}`);
});
```

### ステップ3: クライアント側の実装

`public/index.html`:

```html
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Socket.IO - ブロードキャスト</title>
  <style>
    * {
      margin: 0;
      padding: 0;
      box-sizing: border-box;
    }
    body {
      font-family: Arial, sans-serif;
      background-color: #f5f5f5;
      padding: 20px;
    }
    .container {
      max-width: 1200px;
      margin: 0 auto;
      display: grid;
      grid-template-columns: 250px 1fr;
      gap: 20px;
    }
    .sidebar {
      background-color: white;
      border-radius: 8px;
      padding: 20px;
      box-shadow: 0 2px 5px rgba(0,0,0,0.1);
      height: fit-content;
    }
    .main {
      background-color: white;
      border-radius: 8px;
      padding: 20px;
      box-shadow: 0 2px 5px rgba(0,0,0,0.1);
    }
    h1 {
      color: #333;
      margin-bottom: 20px;
      font-size: 24px;
      border-bottom: 3px solid #4CAF50;
      padding-bottom: 10px;
    }
    h2 {
      color: #666;
      font-size: 18px;
      margin-bottom: 15px;
    }
    .user-info {
      background-color: #e3f2fd;
      padding: 15px;
      border-radius: 5px;
      margin-bottom: 20px;
    }
    .user-info strong {
      color: #1976d2;
    }
    .online-users {
      list-style: none;
    }
    .online-users li {
      padding: 8px;
      margin: 5px 0;
      background-color: #f8f9fa;
      border-radius: 4px;
      border-left: 3px solid #4CAF50;
    }
    .chat-container {
      display: flex;
      flex-direction: column;
      height: 600px;
    }
    .messages {
      flex: 1;
      overflow-y: auto;
      border: 1px solid #ddd;
      border-radius: 5px;
      padding: 15px;
      margin-bottom: 15px;
      background-color: #fafafa;
    }
    .message {
      margin-bottom: 15px;
      padding: 10px;
      border-radius: 5px;
      background-color: white;
      box-shadow: 0 1px 3px rgba(0,0,0,0.1);
    }
    .message.own {
      background-color: #e3f2fd;
      border-left: 3px solid #2196F3;
    }
    .message.system {
      background-color: #fff3e0;
      border-left: 3px solid #ff9800;
    }
    .message.join {
      background-color: #e8f5e9;
      border-left: 3px solid #4CAF50;
    }
    .message.leave {
      background-color: #ffebee;
      border-left: 3px solid #f44336;
    }
    .message-header {
      display: flex;
      justify-content: space-between;
      margin-bottom: 5px;
    }
    .username {
      font-weight: bold;
      color: #1976d2;
    }
    .timestamp {
      font-size: 12px;
      color: #999;
    }
    .typing-indicator {
      font-size: 14px;
      color: #666;
      font-style: italic;
      padding: 5px;
      min-height: 20px;
    }
    .input-area {
      display: flex;
      gap: 10px;
    }
    input[type="text"] {
      flex: 1;
      padding: 12px;
      border: 1px solid #ddd;
      border-radius: 5px;
      font-size: 14px;
    }
    button {
      padding: 12px 24px;
      background-color: #4CAF50;
      color: white;
      border: none;
      border-radius: 5px;
      cursor: pointer;
      font-size: 14px;
    }
    button:hover {
      background-color: #45a049;
    }
    button.announce {
      background-color: #ff9800;
    }
    button.announce:hover {
      background-color: #f57c00;
    }
    .stats {
      background-color: #f8f9fa;
      padding: 15px;
      border-radius: 5px;
      margin-top: 20px;
    }
    .stats-item {
      display: flex;
      justify-content: space-between;
      padding: 5px 0;
    }
  </style>
</head>
<body>
  <div class="container">
    <!-- サイドバー -->
    <div class="sidebar">
      <h2>👤 ユーザー情報</h2>
      <div class="user-info">
        <div><strong>ユーザー名:</strong></div>
        <div id="myUsername">-</div>
        <div style="margin-top: 10px;"><strong>Socket ID:</strong></div>
        <div id="mySocketId" style="font-size: 12px; word-break: break-all;">-</div>
      </div>

      <h2>🟢 オンラインユーザー (<span id="userCount">0</span>)</h2>
      <ul id="userList" class="online-users"></ul>

      <div class="stats">
        <h2>📊 統計</h2>
        <div class="stats-item">
          <span>送信メッセージ:</span>
          <span id="sentCount">0</span>
        </div>
        <div class="stats-item">
          <span>受信メッセージ:</span>
          <span id="receivedCount">0</span>
        </div>
      </div>
    </div>

    <!-- メインエリア -->
    <div class="main">
      <h1>💬 ブロードキャストチャット</h1>

      <div class="chat-container">
        <div class="messages" id="messages"></div>
        <div class="typing-indicator" id="typingIndicator"></div>
        <div class="input-area">
          <input
            type="text"
            id="messageInput"
            placeholder="メッセージを入力..."
            autocomplete="off"
          >
          <button onclick="sendMessage()">送信</button>
          <button onclick="sendAnnouncement()" class="announce">アナウンス</button>
        </div>
      </div>
    </div>
  </div>

  <script src="/socket.io/socket.io.js"></script>
  <script>
    const socket = io();
    let myUsername = '';
    let sentCount = 0;
    let receivedCount = 0;
    let typingTimeout;

    // ===== ユーティリティ関数 =====

    function formatTime(timestamp) {
      const date = new Date(timestamp);
      return date.toLocaleTimeString('ja-JP');
    }

    function addMessage(content, type = 'normal') {
      const messagesDiv = document.getElementById('messages');
      const messageDiv = document.createElement('div');
      messageDiv.className = `message ${type}`;
      messageDiv.innerHTML = content;
      messagesDiv.appendChild(messageDiv);
      messagesDiv.scrollTop = messagesDiv.scrollHeight;

      if (type !== 'own') {
        receivedCount++;
        document.getElementById('receivedCount').textContent = receivedCount;
      }
    }

    function updateUserList(users) {
      const userList = document.getElementById('userList');
      userList.innerHTML = '';
      users.forEach(user => {
        const li = document.createElement('li');
        li.textContent = user.username;
        if (user.id === socket.id) {
          li.textContent += ' (あなた)';
          li.style.fontWeight = 'bold';
        }
        userList.appendChild(li);
      });
    }

    function updateStats() {
      document.getElementById('sentCount').textContent = sentCount;
      document.getElementById('receivedCount').textContent = receivedCount;
    }

    // ===== イベントリスナー =====

    socket.on('connect', () => {
      document.getElementById('mySocketId').textContent = socket.id;
      socket.emit('requestUserList');
    });

    socket.on('userJoined', (data) => {
      console.log('ユーザー参加:', data);

      if (data.user.id === socket.id) {
        // 自分が参加
        myUsername = data.user.username;
        document.getElementById('myUsername').textContent = myUsername;
        addMessage(`
          <div><strong>システム:</strong> チャットに参加しました</div>
          <div class="timestamp">${formatTime(data.timestamp)}</div>
        `, 'system');
      } else {
        // 他のユーザーが参加
        addMessage(`
          <div><strong>${data.user.username}</strong> が参加しました</div>
          <div class="timestamp">${formatTime(data.timestamp)}</div>
        `, 'join');
      }

      document.getElementById('userCount').textContent = data.totalUsers;
      socket.emit('requestUserList');
    });

    socket.on('userLeft', (data) => {
      addMessage(`
        <div><strong>${data.user.username}</strong> が退出しました</div>
        <div class="timestamp">${formatTime(data.timestamp)}</div>
      `, 'leave');

      document.getElementById('userCount').textContent = data.totalUsers;
      socket.emit('requestUserList');
    });

    socket.on('chatMessage', (data) => {
      addMessage(`
        <div class="message-header">
          <span class="username">${data.username}</span>
          <span class="timestamp">${formatTime(data.timestamp)}</span>
        </div>
        <div>${data.message}</div>
      `);
    });

    socket.on('systemAnnouncement', (data) => {
      addMessage(`
        <div class="message-header">
          <span class="username">📢 ${data.from}</span>
          <span class="timestamp">${formatTime(data.timestamp)}</span>
        </div>
        <div>${data.message}</div>
      `, 'system');
    });

    socket.on('userList', (users) => {
      updateUserList(users);
    });

    socket.on('userTyping', (data) => {
      const indicator = document.getElementById('typingIndicator');
      if (data.isTyping) {
        indicator.textContent = `${data.username} が入力中...`;
      } else {
        indicator.textContent = '';
      }
    });

    socket.on('reactionReceived', (data) => {
      console.log('リアクション受信:', data);
    });

    // ===== ユーザーアクション =====

    function sendMessage() {
      const input = document.getElementById('messageInput');
      const message = input.value.trim();

      if (!message) return;

      // サーバーにメッセージ送信
      socket.emit('chatMessage', {
        username: myUsername,
        message: message
      });

      // 自分のメッセージを表示
      addMessage(`
        <div class="message-header">
          <span class="username">${myUsername} (あなた)</span>
          <span class="timestamp">${formatTime(new Date())}</span>
        </div>
        <div>${message}</div>
      `, 'own');

      sentCount++;
      updateStats();

      input.value = '';
      socket.emit('stopTyping', { username: myUsername });
    }

    function sendAnnouncement() {
      const message = prompt('アナウンスメッセージを入力してください:');
      if (!message) return;

      socket.emit('announcement', { message: message });
    }

    // タイピングインジケーター
    document.getElementById('messageInput').addEventListener('input', () => {
      socket.emit('typing', { username: myUsername });

      clearTimeout(typingTimeout);
      typingTimeout = setTimeout(() => {
        socket.emit('stopTyping', { username: myUsername });
      }, 1000);
    });

    // Enterキーで送信
    document.getElementById('messageInput').addEventListener('keypress', (e) => {
      if (e.key === 'Enter') {
        sendMessage();
      }
    });
  </script>
</body>
</html>
```

---

## 🔍 ブロードキャストのパターン

### 1. 全クライアントに送信

```javascript
// サーバー側
io.emit('eventName', data);

// すべての接続クライアントに送信される
```

### 2. 送信元以外に送信

```javascript
// サーバー側
socket.broadcast.emit('eventName', data);

// socket以外のすべてのクライアントに送信
```

### 3. 特定のクライアントにのみ送信

```javascript
// サーバー側
socket.emit('eventName', data);

// このsocketのクライアントにのみ送信
```

### 4. Socket IDを指定して送信

```javascript
// サーバー側
io.to(socketId).emit('eventName', data);

// 特定のSocket IDのクライアントに送信
```

### 比較表

| メソッド | 送信先 | 用途 |
|---------|--------|------|
| `io.emit()` | 全クライアント | システムアナウンス、全体通知 |
| `socket.broadcast.emit()` | 送信元以外 | チャットメッセージ、ステータス更新 |
| `socket.emit()` | 送信元のみ | 個別応答、エラーメッセージ |
| `io.to(id).emit()` | 特定のクライアント | ダイレクトメッセージ、個別通知 |

---

## 🧪 動作確認

### テスト手順

1. サーバーを起動: `node server.js`
2. ブラウザで `http://localhost:3000` を複数タブ/ウィンドウで開く
3. 各タブに異なるユーザー名が表示されることを確認
4. 1つのタブでメッセージを送信
5. 他のタブにメッセージが表示されることを確認
6. オンラインユーザーリストが更新されることを確認

### 期待される動作

```
タブA: ユーザー1として接続
  → タブB、タブCに「ユーザー1が参加しました」と表示

タブA: 「こんにちは」と送信
  → タブA: 自分のメッセージとして表示（青背景）
  → タブB、タブC: ユーザー1からのメッセージとして表示（白背景）

タブBを閉じる
  → タブA、タブC: 「ユーザー2が退出しました」と表示
```

---

## 🐛 よくある問題

### 問題1: 自分にもメッセージが届いてしまう

```javascript
// ❌ 悪い例
io.emit('chat', message);  // 自分にも送信される

// ✅ 良い例
socket.broadcast.emit('chat', message);  // 自分以外に送信
socket.emit('chatSent', message);  // 自分には別イベントで送信
```

### 問題2: 大量のクライアントでパフォーマンス低下

```javascript
// ✅ 解決策: スロットリング
let lastBroadcast = 0;
const THROTTLE_MS = 100;

socket.on('update', (data) => {
  const now = Date.now();
  if (now - lastBroadcast > THROTTLE_MS) {
    socket.broadcast.emit('update', data);
    lastBroadcast = now;
  }
});
```

### 問題3: メモリリーク（ユーザーリストの管理）

```javascript
// ✅ 確実に削除
socket.on('disconnect', () => {
  users.delete(socket.id);  // Mapから削除
  // または
  delete users[socket.id];  // オブジェクトから削除
});
```

---

## 📝 練習課題

### 初級
1. **未読カウンター**: 新しいメッセージを受信したら未読数を表示
2. **オンライン表示**: ユーザーリストに緑の●を表示

### 中級
3. **既読機能**: メッセージを既読したら送信者に通知
4. **リアクション機能**: メッセージに「いいね」を送る

### 上級
5. **メッセージ履歴**: 新規参加者に過去のメッセージを送信
6. **プレゼンス機能**: オンライン/離席/退席のステータス管理

---

## 💡 次のステップ

ブロードキャストの基本ができたら、次は「ルーム」を使ってグループごとに配信する方法を学びましょう！

👉 [Lesson 5: ルーム](./05-rooms.md)

---

## 📚 参考リソース

- [Socket.IO - Broadcasting events](https://socket.io/docs/v4/broadcasting-events/)
- [Socket.IO - Emit cheatsheet](https://socket.io/docs/v4/emit-cheatsheet/)
