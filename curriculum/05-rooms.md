# Lesson 5: ルーム

## 🎯 学習目標

- ルームの概念を理解する
- クライアントをルームに参加・退出させる
- 特定のルームにのみメッセージを送信する
- 複数ルームの管理方法を学ぶ

---

## 📋 メリット

✅ **グループ化**: クライアントを論理的なグループに分ける
✅ **効率的な配信**: 特定のグループにのみメッセージ送信
✅ **スケーラビリティ**: 多数のクライアントを効率的に管理
✅ **柔軟性**: 動的にルームの作成・削除が可能
✅ **プライバシー**: グループ外にメッセージが漏れない

---

## ⚠️ デメリット

❌ **メモリ使用**: ルーム情報の保存にメモリを消費
❌ **管理の複雑さ**: 多数のルームの管理が煩雑になる可能性
❌ **デバッグの難しさ**: どのクライアントがどのルームにいるか追跡が必要
❌ **永続化なし**: サーバー再起動でルーム情報が消える（別途保存が必要）

---

## ⚙️ 技術的原理

### ルームの仕組み

```
ルームA                    ルームB
┌─────────────┐          ┌─────────────┐
│ クライアント1 │          │ クライアント3 │
│ クライアント2 │          │ クライアント4 │
└─────────────┘          └─────────────┘

io.to('roomA').emit()    io.to('roomB').emit()
    ↓                         ↓
クライアント1、2に送信    クライアント3、4に送信
```

### 内部データ構造

Socket.IOは内部でルーム情報をMapとして管理：

```javascript
// 内部構造（簡略版）
{
  rooms: Map {
    'roomA' => Set { 'socketId1', 'socketId2' },
    'roomB' => Set { 'socketId3', 'socketId4' },
    'socketId1' => Set { 'socketId1' }  // 各ソケットは自分のIDをルームとして持つ
  }
}
```

### ルーム参加のフロー

```
クライアント              サーバー
    │                       │
    │  join('roomA')        │
    ├──────────────────────►│
    │                       │ 1. socket.rooms に 'roomA' を追加
    │                       │ 2. rooms['roomA'] に socket.id を追加
    │                       │ 3. 成功
    │                       │
    │  メッセージ送信       │
    │◄──────────────────────┤
    │                       │
```

### パフォーマンス特性

- **参加/退出**: O(1) - 高速
- **ルーム内配信**: O(n) - nはルーム内のクライアント数
- **ルーム一覧取得**: O(1)
- **メモリ**: クライアント数 × 所属ルーム数 に比例

---

## 💼 ユースケース

### 適している場面

1. **チャットルーム**: トピックやチャンネルごとの会話
2. **ゲームロビー**: ゲームルームごとのプレイヤー管理
3. **コラボレーション**: ドキュメントやプロジェクトごとのグループ
4. **通知グループ**: 部署、チーム、役職ごとの通知配信
5. **地域別配信**: 地域やタイムゾーンごとのメッセージ配信

### 注意が必要な場面

1. **1対1通信**: ルームではなくSocket IDで直接送信する方が簡単
2. **全体配信**: ルームを使わず `io.emit()` の方がシンプル
3. **短命なグループ**: 頻繁な作成・削除はオーバーヘッドになる可能性

---

## 💻 実装例

### ステップ1: プロジェクトのセットアップ

```bash
mkdir socket-rooms
cd socket-rooms
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

// ルームごとのユーザー情報を保存
const roomUsers = new Map();

// ヘルパー関数: ルームのユーザーリストを取得
function getRoomUsers(room) {
  if (!roomUsers.has(room)) {
    roomUsers.set(room, new Map());
  }
  return roomUsers.get(room);
}

// ヘルパー関数: ルーム情報を送信
function emitRoomInfo(room) {
  const users = Array.from(getRoomUsers(room).values());
  io.to(room).emit('roomInfo', {
    room: room,
    userCount: users.length,
    users: users
  });
}

io.on('connection', (socket) => {
  console.log(`✅ 接続: ${socket.id}`);

  // ===== ルームへの参加 =====

  socket.on('joinRoom', (data) => {
    const { room, username } = data;

    console.log(`🚪 ${username} がルーム「${room}」に参加`);

    // ルームに参加
    socket.join(room);

    // ユーザー情報を保存
    const users = getRoomUsers(room);
    users.set(socket.id, {
      id: socket.id,
      username: username,
      joinedAt: new Date().toISOString()
    });

    // ルーム内の他のメンバーに通知
    socket.to(room).emit('userJoined', {
      username: username,
      room: room,
      timestamp: new Date().toISOString()
    });

    // 参加者本人に成功を通知
    socket.emit('joinedRoom', {
      room: room,
      message: `ルーム「${room}」に参加しました`
    });

    // ルーム情報を全員に送信
    emitRoomInfo(room);

    console.log(`   現在の参加者: ${users.size}人`);
  });

  // ===== ルームからの退出 =====

  socket.on('leaveRoom', (data) => {
    const { room, username } = data;

    console.log(`🚪 ${username} がルーム「${room}」から退出`);

    // ルームから退出
    socket.leave(room);

    // ユーザー情報を削除
    const users = getRoomUsers(room);
    users.delete(socket.id);

    // ルーム内の残りのメンバーに通知
    socket.to(room).emit('userLeft', {
      username: username,
      room: room,
      timestamp: new Date().toISOString()
    });

    // ルーム情報を更新
    emitRoomInfo(room);

    // ルームが空になったら削除
    if (users.size === 0) {
      roomUsers.delete(room);
      console.log(`   ルーム「${room}」を削除（参加者0）`);
    }
  });

  // ===== ルーム内メッセージ送信 =====

  socket.on('roomMessage', (data) => {
    const { room, username, message } = data;

    console.log(`💬 [${room}] ${username}: ${message}`);

    // ルーム内の他のメンバーにメッセージを送信
    socket.to(room).emit('roomMessage', {
      username: username,
      message: message,
      room: room,
      timestamp: new Date().toISOString()
    });
  });

  // ===== 利用可能なルーム一覧を取得 =====

  socket.on('getRooms', () => {
    const rooms = Array.from(roomUsers.keys()).map(room => ({
      name: room,
      userCount: getRoomUsers(room).size
    }));

    socket.emit('roomList', rooms);
  });

  // ===== 特定のルーム情報を取得 =====

  socket.on('getRoomInfo', (room) => {
    const users = Array.from(getRoomUsers(room).values());
    socket.emit('roomInfo', {
      room: room,
      userCount: users.length,
      users: users
    });
  });

  // ===== 複数ルームへの同時参加 =====

  socket.on('joinMultipleRooms', (data) => {
    const { rooms, username } = data;

    rooms.forEach(room => {
      socket.join(room);

      const users = getRoomUsers(room);
      users.set(socket.id, {
        id: socket.id,
        username: username,
        joinedAt: new Date().toISOString()
      });

      socket.to(room).emit('userJoined', {
        username: username,
        room: room,
        timestamp: new Date().toISOString()
      });
    });

    socket.emit('joinedMultipleRooms', {
      rooms: rooms,
      message: `${rooms.length}個のルームに参加しました`
    });
  });

  // ===== 切断処理 =====

  socket.on('disconnect', () => {
    console.log(`❌ 切断: ${socket.id}`);

    // すべてのルームから削除
    roomUsers.forEach((users, room) => {
      if (users.has(socket.id)) {
        const user = users.get(socket.id);
        users.delete(socket.id);

        // ルームメンバーに通知
        socket.to(room).emit('userLeft', {
          username: user.username,
          room: room,
          timestamp: new Date().toISOString()
        });

        // ルーム情報を更新
        emitRoomInfo(room);

        // 空のルームを削除
        if (users.size === 0) {
          roomUsers.delete(room);
        }
      }
    });
  });

  // ===== デバッグ用 =====

  socket.on('getMyRooms', () => {
    // 現在参加しているルーム一覧
    const rooms = Array.from(socket.rooms).filter(r => r !== socket.id);
    socket.emit('myRooms', rooms);
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
  <title>Socket.IO - ルーム</title>
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
      max-width: 1400px;
      margin: 0 auto;
      display: grid;
      grid-template-columns: 300px 1fr;
      gap: 20px;
    }
    .sidebar {
      display: flex;
      flex-direction: column;
      gap: 20px;
    }
    .card {
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
    .room-list {
      list-style: none;
    }
    .room-item {
      padding: 12px;
      margin: 8px 0;
      background-color: #f8f9fa;
      border-radius: 5px;
      cursor: pointer;
      border-left: 3px solid #4CAF50;
      display: flex;
      justify-content: space-between;
      align-items: center;
    }
    .room-item:hover {
      background-color: #e9ecef;
    }
    .room-item.active {
      background-color: #d4edda;
      border-left-color: #28a745;
      font-weight: bold;
    }
    .user-count {
      background-color: #4CAF50;
      color: white;
      padding: 3px 8px;
      border-radius: 12px;
      font-size: 12px;
    }
    .create-room {
      display: flex;
      gap: 10px;
      margin-bottom: 15px;
    }
    input[type="text"] {
      flex: 1;
      padding: 10px;
      border: 1px solid #ddd;
      border-radius: 5px;
      font-size: 14px;
    }
    button {
      padding: 10px 20px;
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
    button.leave {
      background-color: #f44336;
    }
    button.leave:hover {
      background-color: #da190b;
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
      margin-bottom: 12px;
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
      font-style: italic;
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
    .input-area {
      display: flex;
      gap: 10px;
    }
    .current-room {
      background-color: #e3f2fd;
      padding: 15px;
      border-radius: 5px;
      margin-bottom: 15px;
      border-left: 4px solid #2196F3;
    }
    .user-list {
      list-style: none;
    }
    .user-list li {
      padding: 8px;
      margin: 5px 0;
      background-color: #f8f9fa;
      border-radius: 4px;
      border-left: 3px solid #4CAF50;
    }
  </style>
</head>
<body>
  <div class="container">
    <!-- サイドバー -->
    <div class="sidebar">
      <!-- ユーザー情報 -->
      <div class="card">
        <h2>👤 ユーザー情報</h2>
        <div style="margin-bottom: 10px;">
          <input type="text" id="usernameInput" placeholder="ユーザー名を入力" value="ユーザー1">
        </div>
        <div style="font-size: 12px; color: #666;">
          <strong>Socket ID:</strong><br>
          <span id="socketId" style="word-break: break-all;">-</span>
        </div>
      </div>

      <!-- ルーム作成 -->
      <div class="card">
        <h2>🚪 ルーム作成・参加</h2>
        <div class="create-room">
          <input type="text" id="roomInput" placeholder="ルーム名">
          <button onclick="joinRoom()">参加</button>
        </div>
        <div style="margin-top: 10px;">
          <button onclick="createQuickRoom('ロビー')">ロビーに参加</button>
          <button onclick="createQuickRoom('雑談')">雑談に参加</button>
        </div>
      </div>

      <!-- 現在のルーム -->
      <div class="card">
        <h2>📍 現在のルーム</h2>
        <div id="currentRoomInfo" class="current-room">
          <div><strong>ルーム:</strong> <span id="currentRoomName">未参加</span></div>
          <div><strong>参加者数:</strong> <span id="roomUserCount">0</span></div>
          <div style="margin-top: 10px;">
            <button onclick="leaveRoom()" class="leave">退出</button>
            <button onclick="getRoomInfo()">情報更新</button>
          </div>
        </div>
        <h3 style="margin-top: 15px; font-size: 14px;">メンバー:</h3>
        <ul id="roomMembers" class="user-list"></ul>
      </div>

      <!-- 利用可能なルーム -->
      <div class="card">
        <h2>🌐 利用可能なルーム</h2>
        <button onclick="refreshRooms()" style="width: 100%; margin-bottom: 10px;">
          リスト更新
        </button>
        <ul id="availableRooms" class="room-list"></ul>
      </div>
    </div>

    <!-- メインエリア -->
    <div class="card">
      <h1>💬 ルームチャット</h1>

      <div class="chat-container">
        <div class="messages" id="messages"></div>
        <div class="input-area">
          <input
            type="text"
            id="messageInput"
            placeholder="ルームに参加してメッセージを送信..."
            disabled
          >
          <button onclick="sendMessage()" id="sendButton" disabled>送信</button>
        </div>
      </div>
    </div>
  </div>

  <script src="/socket.io/socket.io.js"></script>
  <script>
    const socket = io();
    let currentRoom = null;
    let myUsername = 'ユーザー1';

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
    }

    function updateRoomUI(room) {
      currentRoom = room;
      document.getElementById('currentRoomName').textContent = room || '未参加';

      const messageInput = document.getElementById('messageInput');
      const sendButton = document.getElementById('sendButton');

      if (room) {
        messageInput.disabled = false;
        messageInput.placeholder = `${room}にメッセージを送信...`;
        sendButton.disabled = false;
      } else {
        messageInput.disabled = true;
        messageInput.placeholder = 'ルームに参加してメッセージを送信...';
        sendButton.disabled = true;
      }
    }

    // ===== イベントリスナー =====

    socket.on('connect', () => {
      document.getElementById('socketId').textContent = socket.id;
      addMessage(`<div><strong>システム:</strong> サーバーに接続しました</div>`, 'system');
      refreshRooms();
    });

    socket.on('joinedRoom', (data) => {
      addMessage(`
        <div><strong>システム:</strong> ${data.message}</div>
      `, 'system');
      updateRoomUI(data.room);
    });

    socket.on('userJoined', (data) => {
      addMessage(`
        <div class="message-header">
          <span><strong>${data.username}</strong> が参加しました</span>
          <span class="timestamp">${formatTime(data.timestamp)}</span>
        </div>
      `, 'system');
    });

    socket.on('userLeft', (data) => {
      addMessage(`
        <div class="message-header">
          <span><strong>${data.username}</strong> が退出しました</span>
          <span class="timestamp">${formatTime(data.timestamp)}</span>
        </div>
      `, 'system');
    });

    socket.on('roomMessage', (data) => {
      addMessage(`
        <div class="message-header">
          <span class="username">${data.username}</span>
          <span class="timestamp">${formatTime(data.timestamp)}</span>
        </div>
        <div>${data.message}</div>
      `);
    });

    socket.on('roomInfo', (data) => {
      document.getElementById('roomUserCount').textContent = data.userCount;

      const membersList = document.getElementById('roomMembers');
      membersList.innerHTML = '';
      data.users.forEach(user => {
        const li = document.createElement('li');
        li.textContent = user.username;
        if (user.id === socket.id) {
          li.textContent += ' (あなた)';
          li.style.fontWeight = 'bold';
        }
        membersList.appendChild(li);
      });
    });

    socket.on('roomList', (rooms) => {
      const roomList = document.getElementById('availableRooms');
      roomList.innerHTML = '';

      if (rooms.length === 0) {
        roomList.innerHTML = '<li style="padding: 10px; color: #999;">利用可能なルームはありません</li>';
        return;
      }

      rooms.forEach(room => {
        const li = document.createElement('li');
        li.className = 'room-item';
        if (room.name === currentRoom) {
          li.classList.add('active');
        }
        li.innerHTML = `
          <span>${room.name}</span>
          <span class="user-count">${room.userCount}人</span>
        `;
        li.onclick = () => {
          document.getElementById('roomInput').value = room.name;
        };
        roomList.appendChild(li);
      });
    });

    // ===== ユーザーアクション =====

    function joinRoom() {
      const roomInput = document.getElementById('roomInput');
      const room = roomInput.value.trim();
      const usernameInput = document.getElementById('usernameInput');
      myUsername = usernameInput.value.trim() || 'ユーザー1';

      if (!room) {
        alert('ルーム名を入力してください');
        return;
      }

      if (currentRoom) {
        if (currentRoom === room) {
          alert('すでにこのルームに参加しています');
          return;
        }
        // 現在のルームから退出
        socket.emit('leaveRoom', { room: currentRoom, username: myUsername });
      }

      socket.emit('joinRoom', { room: room, username: myUsername });
      roomInput.value = '';
      refreshRooms();
    }

    function leaveRoom() {
      if (!currentRoom) {
        alert('ルームに参加していません');
        return;
      }

      socket.emit('leaveRoom', { room: currentRoom, username: myUsername });
      updateRoomUI(null);
      refreshRooms();
    }

    function sendMessage() {
      if (!currentRoom) {
        alert('ルームに参加してください');
        return;
      }

      const input = document.getElementById('messageInput');
      const message = input.value.trim();

      if (!message) return;

      socket.emit('roomMessage', {
        room: currentRoom,
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

      input.value = '';
    }

    function createQuickRoom(roomName) {
      document.getElementById('roomInput').value = roomName;
      joinRoom();
    }

    function refreshRooms() {
      socket.emit('getRooms');
    }

    function getRoomInfo() {
      if (currentRoom) {
        socket.emit('getRoomInfo', currentRoom);
      }
    }

    // Enterキーで送信
    document.getElementById('messageInput').addEventListener('keypress', (e) => {
      if (e.key === 'Enter') {
        sendMessage();
      }
    });

    document.getElementById('roomInput').addEventListener('keypress', (e) => {
      if (e.key === 'Enter') {
        joinRoom();
      }
    });
  </script>
</body>
</html>
```

---

## 🔍 ルームAPIの詳細

### 基本操作

```javascript
// ルームに参加
socket.join('roomName');

// ルームから退出
socket.leave('roomName');

// 複数のルームに参加
socket.join(['room1', 'room2', 'room3']);

// 現在参加しているルーム一覧を取得
const rooms = Array.from(socket.rooms);
// 注: socket.rooms には socket.id も含まれる
```

### メッセージ送信

```javascript
// 特定のルームに送信
io.to('roomName').emit('event', data);

// 複数のルームに送信
io.to('room1').to('room2').emit('event', data);

// ルームに送信（送信元を除く）
socket.to('roomName').emit('event', data);

// 自分が参加しているすべてのルームに送信
socket.rooms.forEach(room => {
  if (room !== socket.id) {
    io.to(room).emit('event', data);
  }
});
```

### ルーム情報の取得

```javascript
// サーバー側でルームの全ソケットを取得
const socketsInRoom = await io.in('roomName').fetchSockets();
console.log(`ルーム内のクライアント数: ${socketsInRoom.length}`);

// ルーム内の各ソケットを処理
socketsInRoom.forEach(socket => {
  console.log(socket.id);
  console.log(socket.handshake);
});

// すべてのルームを取得
const rooms = io.sockets.adapter.rooms;
```

---

## 🧪 動作確認

### テスト手順

1. サーバーを起動: `node server.js`
2. ブラウザで複数タブを開く: `http://localhost:3000`
3. タブA: 「ロビー」に参加
4. タブB: 「ロビー」に参加
5. タブC: 「雑談」に参加
6. タブAでメッセージ送信 → タブBのみ受信（タブCは受信しない）
7. 「利用可能なルーム」に「ロビー（2人）」「雑談（1人）」と表示されることを確認

---

## 🐛 よくある問題

### 問題1: ルームから退出したのにメッセージが届く

```javascript
// ❌ 悪い例: 退出を忘れる
socket.on('switchRoom', (newRoom) => {
  socket.join(newRoom);  // 新しいルームに参加するだけ
});

// ✅ 良い例: 既存のルームから退出
socket.on('switchRoom', (data) => {
  if (data.oldRoom) {
    socket.leave(data.oldRoom);
  }
  socket.join(data.newRoom);
});
```

### 問題2: ルーム一覧の管理

```javascript
// ✅ カスタムルーム一覧を管理
const activeRooms = new Map();

socket.on('joinRoom', (room) => {
  socket.join(room);

  if (!activeRooms.has(room)) {
    activeRooms.set(room, new Set());
  }
  activeRooms.get(room).add(socket.id);
});

socket.on('disconnect', () => {
  activeRooms.forEach((users, room) => {
    users.delete(socket.id);
    if (users.size === 0) {
      activeRooms.delete(room);
    }
  });
});
```

### 問題3: メモリリーク

```javascript
// ✅ 切断時にすべてのルームから削除
socket.on('disconnect', () => {
  // Socket.IOが自動的に処理するが、カスタムデータは手動削除
  roomUsers.forEach((users, room) => {
    users.delete(socket.id);
  });
});
```

---

## 📝 練習課題

### 初級
1. **プライベートルーム**: パスワード付きルームを実装
2. **最大人数制限**: ルームの最大参加者数を設定

### 中級
3. **ルーム管理者**: 最初に参加したユーザーを管理者に設定
4. **招待機能**: 他のユーザーを特定のルームに招待

### 上級
5. **一時的なルーム**: 一定時間後に自動削除されるルーム
6. **ルーム統計**: 各ルームのメッセージ数、アクティビティを記録

---

## 💡 次のステップ

ルームの使い方を理解したら、次は「ネームスペース」を学んで、アプリケーションをさらに大きく分割する方法を学びましょう！

👉 [Lesson 6: ネームスペース](./06-namespaces.md)

---

## 📚 参考リソース

- [Socket.IO - Rooms](https://socket.io/docs/v4/rooms/)
- [Socket.IO - Server API - join](https://socket.io/docs/v4/server-api/#socketjoinroom)
