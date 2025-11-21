# Lesson 6: ネームスペース

## 🎯 学習目標

- ネームスペースの概念を理解する
- カスタムネームスペースを作成・使用する
- ネームスペースとルームの違いを理解する
- 動的ネームスペースを実装する

---

## 📋 メリット

✅ **論理的分離**: アプリケーション機能を完全に分離
✅ **独立した接続**: 各ネームスペースで独自の接続を持つ
✅ **ミドルウェア**: ネームスペースごとに異なる認証・処理を適用
✅ **スケーラビリティ**: 機能ごとに独立したインスタンスを管理
✅ **セキュリティ**: 機能ごとにアクセス制御を実装

---

## ⚠️ デメリット

❌ **接続オーバーヘッド**: 各ネームスペースで独立した接続が必要
❌ **リソース消費**: ネームスペース数に応じてメモリ・CPU消費増加
❌ **複雑性**: ルームで十分な場合もある
❌ **クライアント管理**: 複数ネームスペースへの接続管理が必要

---

## ⚙️ 技術的原理

### ネームスペースの構造

```
サーバー (io)
│
├── / (デフォルトネームスペース)
│   ├── クライアントA
│   └── クライアントB
│
├── /chat (チャット用ネームスペース)
│   ├── クライアントC
│   └── クライアントD
│
└── /admin (管理者用ネームスペース)
    └── クライアントE
```

### ネームスペースとルームの違い

```
ネームスペース: アプリケーション全体の大きな区分
└── ルーム: ネームスペース内での小さなグループ

例:
/chat (ネームスペース)
  ├── room1 (ルーム)
  │   ├── ユーザーA
  │   └── ユーザーB
  └── room2 (ルーム)
      ├── ユーザーC
      └── ユーザーD
```

### 内部動作

```javascript
// ネームスペース作成時の内部処理
1. 新しい Namespace インスタンスを作成
2. 独自のイベントリスナーを設定
3. 独自のミドルウェアスタックを持つ
4. 独自の Adapter を持つ（ルーム管理用）

// クライアント接続時
1. URLのパス部分を解析 (/chat など)
2. 該当するネームスペースを検索
3. ネームスペースのミドルウェアを実行
4. 接続を確立
```

### パフォーマンス特性

- **メモリ**: ネームスペース数 + クライアント数 に比例
- **接続確立**: ネームスペースごとに独立したハンドシェイク
- **メッセージ配信**: ネームスペース内でのみ配信されるため効率的

---

## 💼 ユースケース

### 適している場面

1. **マルチテナント**: 企業ごとに完全に分離されたチャンネル
2. **機能分離**: チャット、通知、ダッシュボードを完全に分離
3. **権限管理**: 管理者用と一般ユーザー用で異なるネームスペース
4. **API バージョニング**: `/v1`, `/v2` など異なるAPIバージョン
5. **異なるプロトコル**: ビジネスロジックごとに異なるイベント体系

### ルームで十分な場面

1. **同じアプリ内でのグループ化**: チャットルーム、ゲームルーム
2. **一時的なグループ**: セッションベースのグループ
3. **シンプルな分類**: カテゴリやトピックによる分類

---

## 💻 実装例

### ステップ1: プロジェクトのセットアップ

```bash
mkdir socket-namespaces
cd socket-namespaces
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

// ===== デフォルトネームスペース (/) =====

io.on('connection', (socket) => {
  console.log(`[/] 接続: ${socket.id}`);

  socket.emit('welcome', {
    message: 'デフォルトネームスペースに接続しました',
    namespace: '/'
  });

  socket.on('message', (data) => {
    console.log(`[/] メッセージ: ${data}`);
    socket.broadcast.emit('message', data);
  });

  socket.on('disconnect', () => {
    console.log(`[/] 切断: ${socket.id}`);
  });
});

// ===== チャット用ネームスペース (/chat) =====

const chatNamespace = io.of('/chat');

chatNamespace.on('connection', (socket) => {
  console.log(`[/chat] 接続: ${socket.id}`);

  socket.emit('welcome', {
    message: 'チャットネームスペースに接続しました',
    namespace: '/chat'
  });

  socket.on('joinRoom', (room) => {
    socket.join(room);
    console.log(`[/chat] ${socket.id} がルーム ${room} に参加`);

    socket.to(room).emit('userJoined', {
      socketId: socket.id,
      room: room,
      timestamp: new Date().toISOString()
    });

    socket.emit('roomJoined', { room: room });
  });

  socket.on('chatMessage', (data) => {
    console.log(`[/chat] [${data.room}] ${data.username}: ${data.message}`);

    socket.to(data.room).emit('chatMessage', {
      username: data.username,
      message: data.message,
      timestamp: new Date().toISOString()
    });
  });

  socket.on('disconnect', () => {
    console.log(`[/chat] 切断: ${socket.id}`);
  });
});

// ===== 管理者用ネームスペース (/admin) =====

const adminNamespace = io.of('/admin');

// ミドルウェア: 認証チェック
adminNamespace.use((socket, next) => {
  const token = socket.handshake.auth.token;

  console.log(`[/admin] 認証チェック: ${token}`);

  // 簡易的な認証（実際はJWTなどを使用）
  if (token === 'admin-secret-token') {
    next();
  } else {
    next(new Error('認証失敗'));
  }
});

adminNamespace.on('connection', (socket) => {
  console.log(`[/admin] 接続: ${socket.id}`);

  socket.emit('welcome', {
    message: '管理者ネームスペースに接続しました',
    namespace: '/admin'
  });

  // 統計情報を送信
  socket.on('getStats', async () => {
    const defaultSockets = await io.fetchSockets();
    const chatSockets = await chatNamespace.fetchSockets();
    const adminSockets = await adminNamespace.fetchSockets();

    socket.emit('stats', {
      default: defaultSockets.length,
      chat: chatSockets.length,
      admin: adminSockets.length,
      total: defaultSockets.length + chatSockets.length + adminSockets.length
    });
  });

  // 全体アナウンス（全ネームスペースに送信）
  socket.on('broadcastAll', (message) => {
    console.log(`[/admin] 全体アナウンス: ${message}`);

    io.emit('announcement', {
      from: 'Admin',
      message: message,
      timestamp: new Date().toISOString()
    });

    chatNamespace.emit('announcement', {
      from: 'Admin',
      message: message,
      timestamp: new Date().toISOString()
    });
  });

  socket.on('disconnect', () => {
    console.log(`[/admin] 切断: ${socket.id}`);
  });
});

// ===== 通知用ネームスペース (/notifications) =====

const notificationNamespace = io.of('/notifications');

notificationNamespace.on('connection', (socket) => {
  console.log(`[/notifications] 接続: ${socket.id}`);

  socket.emit('welcome', {
    message: '通知ネームスペースに接続しました',
    namespace: '/notifications'
  });

  // ユーザーIDでルームに参加（個別通知用）
  socket.on('registerUser', (userId) => {
    socket.join(`user:${userId}`);
    console.log(`[/notifications] ユーザー ${userId} を登録`);

    socket.emit('registered', { userId: userId });
  });

  socket.on('disconnect', () => {
    console.log(`[/notifications] 切断: ${socket.id}`);
  });
});

// 定期的に通知を送信（デモ用）
setInterval(() => {
  notificationNamespace.emit('systemNotification', {
    type: 'info',
    message: 'システムの定期チェックが完了しました',
    timestamp: new Date().toISOString()
  });
}, 30000);

// ===== 動的ネームスペース (/dynamic-*) =====

// 正規表現でマッチするネームスペースを作成
const dynamicNamespace = io.of(/^\/dynamic-\w+$/);

dynamicNamespace.on('connection', (socket) => {
  const namespace = socket.nsp.name;
  console.log(`[${namespace}] 接続: ${socket.id}`);

  socket.emit('welcome', {
    message: `動的ネームスペース ${namespace} に接続しました`,
    namespace: namespace
  });

  socket.on('message', (data) => {
    console.log(`[${namespace}] メッセージ: ${data}`);
    socket.broadcast.emit('message', data);
  });

  socket.on('disconnect', () => {
    console.log(`[${namespace}] 切断: ${socket.id}`);
  });
});

// ===== サーバー起動 =====

const PORT = 3000;
httpServer.listen(PORT, () => {
  console.log(`🚀 サーバー起動: http://localhost:${PORT}`);
  console.log(`利用可能なネームスペース:`);
  console.log(`  - / (デフォルト)`);
  console.log(`  - /chat (チャット)`);
  console.log(`  - /admin (管理者)`);
  console.log(`  - /notifications (通知)`);
  console.log(`  - /dynamic-* (動的)`);
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
  <title>Socket.IO - ネームスペース</title>
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
    }
    h1 {
      color: #333;
      margin-bottom: 20px;
      border-bottom: 3px solid #4CAF50;
      padding-bottom: 10px;
    }
    .namespaces-grid {
      display: grid;
      grid-template-columns: repeat(auto-fit, minmax(400px, 1fr));
      gap: 20px;
      margin-bottom: 20px;
    }
    .namespace-card {
      background-color: white;
      border-radius: 8px;
      padding: 20px;
      box-shadow: 0 2px 5px rgba(0,0,0,0.1);
    }
    .namespace-header {
      display: flex;
      justify-content: space-between;
      align-items: center;
      margin-bottom: 15px;
      padding-bottom: 10px;
      border-bottom: 2px solid #eee;
    }
    .namespace-title {
      font-size: 18px;
      font-weight: bold;
      color: #333;
    }
    .status-badge {
      padding: 5px 12px;
      border-radius: 15px;
      font-size: 12px;
      font-weight: bold;
    }
    .status-connected {
      background-color: #d4edda;
      color: #155724;
    }
    .status-disconnected {
      background-color: #f8d7da;
      color: #721c24;
    }
    .log {
      background-color: #f8f9fa;
      border: 1px solid #ddd;
      border-radius: 5px;
      padding: 10px;
      height: 200px;
      overflow-y: auto;
      margin-bottom: 10px;
      font-family: monospace;
      font-size: 12px;
    }
    .log-entry {
      margin: 3px 0;
      padding: 3px;
    }
    .input-group {
      display: flex;
      gap: 10px;
      margin-bottom: 10px;
    }
    input[type="text"] {
      flex: 1;
      padding: 8px;
      border: 1px solid #ddd;
      border-radius: 4px;
      font-size: 14px;
    }
    button {
      padding: 8px 16px;
      background-color: #4CAF50;
      color: white;
      border: none;
      border-radius: 4px;
      cursor: pointer;
      font-size: 14px;
    }
    button:hover {
      background-color: #45a049;
    }
    button.disconnect {
      background-color: #f44336;
    }
    button.disconnect:hover {
      background-color: #da190b;
    }
    button.secondary {
      background-color: #2196F3;
    }
    button.secondary:hover {
      background-color: #0b7dda;
    }
    .admin-controls {
      background-color: #fff3e0;
      padding: 15px;
      border-radius: 5px;
      margin-top: 10px;
    }
    .stats-grid {
      display: grid;
      grid-template-columns: repeat(4, 1fr);
      gap: 10px;
      margin-top: 10px;
    }
    .stat-card {
      background-color: white;
      padding: 15px;
      border-radius: 5px;
      text-align: center;
      border-left: 4px solid #4CAF50;
    }
    .stat-number {
      font-size: 24px;
      font-weight: bold;
      color: #4CAF50;
    }
    .stat-label {
      font-size: 12px;
      color: #666;
      margin-top: 5px;
    }
  </style>
</head>
<body>
  <div class="container">
    <h1>🌐 Socket.IO - ネームスペース</h1>

    <div class="namespaces-grid">
      <!-- デフォルトネームスペース -->
      <div class="namespace-card">
        <div class="namespace-header">
          <div class="namespace-title">/ (デフォルト)</div>
          <div id="status-default" class="status-badge status-disconnected">未接続</div>
        </div>
        <div id="log-default" class="log"></div>
        <div class="input-group">
          <button onclick="connectNamespace('default')">接続</button>
          <button onclick="disconnectNamespace('default')" class="disconnect">切断</button>
        </div>
        <div class="input-group">
          <input type="text" id="input-default" placeholder="メッセージ">
          <button onclick="sendMessage('default')">送信</button>
        </div>
      </div>

      <!-- チャットネームスペース -->
      <div class="namespace-card">
        <div class="namespace-header">
          <div class="namespace-title">/chat (チャット)</div>
          <div id="status-chat" class="status-badge status-disconnected">未接続</div>
        </div>
        <div id="log-chat" class="log"></div>
        <div class="input-group">
          <button onclick="connectNamespace('chat')">接続</button>
          <button onclick="disconnectNamespace('chat')" class="disconnect">切断</button>
        </div>
        <div class="input-group">
          <input type="text" id="room-chat" placeholder="ルーム名" value="lobby">
          <button onclick="joinChatRoom()" class="secondary">ルーム参加</button>
        </div>
        <div class="input-group">
          <input type="text" id="input-chat" placeholder="メッセージ">
          <button onclick="sendChatMessage()">送信</button>
        </div>
      </div>

      <!-- 管理者ネームスペース -->
      <div class="namespace-card">
        <div class="namespace-header">
          <div class="namespace-title">/admin (管理者)</div>
          <div id="status-admin" class="status-badge status-disconnected">未接続</div>
        </div>
        <div id="log-admin" class="log"></div>
        <div class="input-group">
          <input type="text" id="token-admin" placeholder="トークン" value="admin-secret-token">
          <button onclick="connectNamespace('admin')">接続</button>
          <button onclick="disconnectNamespace('admin')" class="disconnect">切断</button>
        </div>
        <div class="admin-controls">
          <button onclick="getStats()" class="secondary">統計取得</button>
          <button onclick="broadcastAll()">全体アナウンス</button>
          <div id="stats" class="stats-grid" style="display: none; margin-top: 15px;">
            <div class="stat-card">
              <div class="stat-number" id="stat-default">0</div>
              <div class="stat-label">デフォルト</div>
            </div>
            <div class="stat-card">
              <div class="stat-number" id="stat-chat">0</div>
              <div class="stat-label">チャット</div>
            </div>
            <div class="stat-card">
              <div class="stat-number" id="stat-admin">0</div>
              <div class="stat-label">管理者</div>
            </div>
            <div class="stat-card">
              <div class="stat-number" id="stat-total">0</div>
              <div class="stat-label">合計</div>
            </div>
          </div>
        </div>
      </div>

      <!-- 通知ネームスペース -->
      <div class="namespace-card">
        <div class="namespace-header">
          <div class="namespace-title">/notifications (通知)</div>
          <div id="status-notifications" class="status-badge status-disconnected">未接続</div>
        </div>
        <div id="log-notifications" class="log"></div>
        <div class="input-group">
          <button onclick="connectNamespace('notifications')">接続</button>
          <button onclick="disconnectNamespace('notifications')" class="disconnect">切断</button>
        </div>
        <div class="input-group">
          <input type="text" id="userId-notifications" placeholder="ユーザーID" value="user123">
          <button onclick="registerUser()" class="secondary">ユーザー登録</button>
        </div>
      </div>

      <!-- 動的ネームスペース -->
      <div class="namespace-card">
        <div class="namespace-header">
          <div class="namespace-title">/dynamic-* (動的)</div>
          <div id="status-dynamic" class="status-badge status-disconnected">未接続</div>
        </div>
        <div id="log-dynamic" class="log"></div>
        <div class="input-group">
          <input type="text" id="dynamic-name" placeholder="名前 (例: test)" value="test">
          <button onclick="connectDynamic()">接続</button>
          <button onclick="disconnectNamespace('dynamic')" class="disconnect">切断</button>
        </div>
        <div class="input-group">
          <input type="text" id="input-dynamic" placeholder="メッセージ">
          <button onclick="sendMessage('dynamic')">送信</button>
        </div>
      </div>
    </div>
  </div>

  <script src="/socket.io/socket.io.js"></script>
  <script>
    const sockets = {};
    let currentChatRoom = null;

    function addLog(namespace, message) {
      const log = document.getElementById(`log-${namespace}`);
      const entry = document.createElement('div');
      entry.className = 'log-entry';
      const time = new Date().toLocaleTimeString();
      entry.textContent = `[${time}] ${message}`;
      log.appendChild(entry);
      log.scrollTop = log.scrollHeight;
    }

    function updateStatus(namespace, connected) {
      const status = document.getElementById(`status-${namespace}`);
      if (connected) {
        status.textContent = '接続中';
        status.className = 'status-badge status-connected';
      } else {
        status.textContent = '未接続';
        status.className = 'status-badge status-disconnected';
      }
    }

    // デフォルトネームスペース
    function connectNamespace(type) {
      if (type === 'default') {
        if (sockets.default) {
          addLog('default', '既に接続されています');
          return;
        }

        sockets.default = io('/');

        sockets.default.on('connect', () => {
          addLog('default', '接続しました');
          updateStatus('default', true);
        });

        sockets.default.on('welcome', (data) => {
          addLog('default', data.message);
        });

        sockets.default.on('message', (data) => {
          addLog('default', `受信: ${data}`);
        });

        sockets.default.on('announcement', (data) => {
          addLog('default', `📢 ${data.from}: ${data.message}`);
        });

        sockets.default.on('disconnect', () => {
          addLog('default', '切断されました');
          updateStatus('default', false);
        });
      } else if (type === 'chat') {
        if (sockets.chat) {
          addLog('chat', '既に接続されています');
          return;
        }

        sockets.chat = io('/chat');

        sockets.chat.on('connect', () => {
          addLog('chat', '接続しました');
          updateStatus('chat', true);
        });

        sockets.chat.on('welcome', (data) => {
          addLog('chat', data.message);
        });

        sockets.chat.on('roomJoined', (data) => {
          currentChatRoom = data.room;
          addLog('chat', `ルーム「${data.room}」に参加しました`);
        });

        sockets.chat.on('userJoined', (data) => {
          addLog('chat', `ユーザーが参加しました (${data.socketId})`);
        });

        sockets.chat.on('chatMessage', (data) => {
          addLog('chat', `${data.username}: ${data.message}`);
        });

        sockets.chat.on('announcement', (data) => {
          addLog('chat', `📢 ${data.from}: ${data.message}`);
        });

        sockets.chat.on('disconnect', () => {
          addLog('chat', '切断されました');
          updateStatus('chat', false);
          currentChatRoom = null;
        });
      } else if (type === 'admin') {
        if (sockets.admin) {
          addLog('admin', '既に接続されています');
          return;
        }

        const token = document.getElementById('token-admin').value;

        sockets.admin = io('/admin', {
          auth: { token: token }
        });

        sockets.admin.on('connect', () => {
          addLog('admin', '接続しました（認証成功）');
          updateStatus('admin', true);
        });

        sockets.admin.on('connect_error', (err) => {
          addLog('admin', `接続エラー: ${err.message}`);
          updateStatus('admin', false);
          sockets.admin = null;
        });

        sockets.admin.on('welcome', (data) => {
          addLog('admin', data.message);
        });

        sockets.admin.on('stats', (data) => {
          addLog('admin', `統計: デフォルト=${data.default}, チャット=${data.chat}, 管理者=${data.admin}, 合計=${data.total}`);
          document.getElementById('stat-default').textContent = data.default;
          document.getElementById('stat-chat').textContent = data.chat;
          document.getElementById('stat-admin').textContent = data.admin;
          document.getElementById('stat-total').textContent = data.total;
          document.getElementById('stats').style.display = 'grid';
        });

        sockets.admin.on('disconnect', () => {
          addLog('admin', '切断されました');
          updateStatus('admin', false);
        });
      } else if (type === 'notifications') {
        if (sockets.notifications) {
          addLog('notifications', '既に接続されています');
          return;
        }

        sockets.notifications = io('/notifications');

        sockets.notifications.on('connect', () => {
          addLog('notifications', '接続しました');
          updateStatus('notifications', true);
        });

        sockets.notifications.on('welcome', (data) => {
          addLog('notifications', data.message);
        });

        sockets.notifications.on('registered', (data) => {
          addLog('notifications', `ユーザー登録完了: ${data.userId}`);
        });

        sockets.notifications.on('systemNotification', (data) => {
          addLog('notifications', `🔔 ${data.message}`);
        });

        sockets.notifications.on('disconnect', () => {
          addLog('notifications', '切断されました');
          updateStatus('notifications', false);
        });
      }
    }

    function connectDynamic() {
      const name = document.getElementById('dynamic-name').value;
      if (!name) {
        alert('名前を入力してください');
        return;
      }

      if (sockets.dynamic) {
        sockets.dynamic.disconnect();
      }

      sockets.dynamic = io(`/dynamic-${name}`);

      sockets.dynamic.on('connect', () => {
        addLog('dynamic', `/dynamic-${name} に接続しました`);
        updateStatus('dynamic', true);
      });

      sockets.dynamic.on('welcome', (data) => {
        addLog('dynamic', data.message);
      });

      sockets.dynamic.on('message', (data) => {
        addLog('dynamic', `受信: ${data}`);
      });

      sockets.dynamic.on('disconnect', () => {
        addLog('dynamic', '切断されました');
        updateStatus('dynamic', false);
      });
    }

    function disconnectNamespace(type) {
      if (sockets[type]) {
        sockets[type].disconnect();
        sockets[type] = null;
        addLog(type, '切断しました');
        updateStatus(type, false);
      }
    }

    function sendMessage(type) {
      if (!sockets[type]) {
        alert('接続してください');
        return;
      }

      const input = document.getElementById(`input-${type}`);
      const message = input.value.trim();

      if (!message) return;

      sockets[type].emit('message', message);
      addLog(type, `送信: ${message}`);
      input.value = '';
    }

    function joinChatRoom() {
      if (!sockets.chat) {
        alert('チャットネームスペースに接続してください');
        return;
      }

      const room = document.getElementById('room-chat').value.trim();
      if (!room) return;

      sockets.chat.emit('joinRoom', room);
    }

    function sendChatMessage() {
      if (!sockets.chat) {
        alert('チャットネームスペースに接続してください');
        return;
      }

      if (!currentChatRoom) {
        alert('ルームに参加してください');
        return;
      }

      const input = document.getElementById('input-chat');
      const message = input.value.trim();

      if (!message) return;

      sockets.chat.emit('chatMessage', {
        room: currentChatRoom,
        username: 'ユーザー',
        message: message
      });

      addLog('chat', `送信: ${message}`);
      input.value = '';
    }

    function getStats() {
      if (!sockets.admin) {
        alert('管理者ネームスペースに接続してください');
        return;
      }

      sockets.admin.emit('getStats');
    }

    function broadcastAll() {
      if (!sockets.admin) {
        alert('管理者ネームスペースに接続してください');
        return;
      }

      const message = prompt('全体アナウンスメッセージ:');
      if (message) {
        sockets.admin.emit('broadcastAll', message);
        addLog('admin', `全体アナウンス送信: ${message}`);
      }
    }

    function registerUser() {
      if (!sockets.notifications) {
        alert('通知ネームスペースに接続してください');
        return;
      }

      const userId = document.getElementById('userId-notifications').value;
      sockets.notifications.emit('registerUser', userId);
    }
  </script>
</body>
</html>
```

---

## 🔍 ネームスペースAPIの詳細

### サーバー側

```javascript
// カスタムネームスペースを作成
const customNsp = io.of('/custom');

// 動的ネームスペース（正規表現）
const dynamicNsp = io.of(/^\/dynamic-\w+$/);

// ネームスペース内で接続を待機
customNsp.on('connection', (socket) => {
  // socket.nsp でネームスペースにアクセス
  console.log(socket.nsp.name);  // '/custom'
});

// ネームスペース全体にブロードキャスト
customNsp.emit('event', data);

// ネームスペース内の特定のルームに送信
customNsp.to('room1').emit('event', data);

// ネームスペース内のソケット一覧を取得
const sockets = await customNsp.fetchSockets();
```

### クライアント側

```javascript
// デフォルトネームスペース
const socket = io();  // または io('/')

// カスタムネームスペース
const customSocket = io('/custom');

// オプション付き接続
const adminSocket = io('/admin', {
  auth: {
    token: 'secret-token'
  }
});

// 切断
customSocket.disconnect();

// 再接続
customSocket.connect();
```

---

## 🧪 動作確認

### テスト手順

1. サーバー起動: `node server.js`
2. ブラウザで `http://localhost:3000` を開く
3. 各ネームスペースの「接続」ボタンをクリック
4. それぞれ独立して動作することを確認
5. 管理者ネームスペースで統計を取得
6. 全体アナウンスを送信して、他のネームスペースに届くことを確認

---

## 📝 練習課題

### 初級
1. **カスタムネームスペース**: `/game` ネームスペースを作成
2. **接続カウント**: 各ネームスペースの接続数を表示

### 中級
3. **ネームスペースミドルウェア**: 各ネームスペースに異なる認証を実装
4. **クロスネームスペース通信**: あるネームスペースから別のネームスペースにメッセージ送信

### 上級
5. **動的ネームスペース生成**: ユーザーが任意のネームスペースを作成できる機能
6. **ネームスペース統計**: 各ネームスペースのメッセージ数、アクティビティを記録

---

## 💡 次のステップ

ネームスペースの使い方を理解したら、次は「ミドルウェア」を学んで、認証や前処理を実装する方法を学びましょう！

👉 [Lesson 7: ミドルウェア](./07-middleware.md)

---

## 📚 参考リソース

- [Socket.IO - Namespaces](https://socket.io/docs/v4/namespaces/)
- [Socket.IO - Dynamic namespaces](https://socket.io/docs/v4/namespaces/#dynamic-namespaces)
