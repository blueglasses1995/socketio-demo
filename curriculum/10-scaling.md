# Lesson 10: スケーリング

## 🎯 学習目標

- Socket.IOのスケーリングの課題を理解する
- Redis Adapterを使った水平スケーリングを実装する
- 複数サーバー間でのメッセージング を実現する
- Sticky Sessionの設定を理解する

---

## 📋 メリット

✅ **高可用性**: サーバーダウン時も他のサーバーで継続
✅ **負荷分散**: 複数サーバーで負荷を分散
✅ **スケーラビリティ**: ユーザー数に応じてサーバーを増減
✅ **パフォーマンス**: CPU・メモリを効率的に利用
✅ **地理的分散**: 世界中にサーバーを配置してレイテンシを削減

---

## ⚠️ デメリット

❌ **複雑性**: アーキテクチャが複雑になる
❌ **コスト**: Redis等の追加インフラが必要
❌ **デバッグ**: 複数サーバーでのデバッグが困難
❌ **レイテンシ**: Redis経由の通信でわずかな遅延
❌ **Single Point of Failure**: Redisがダウンすると全体が影響

---

## ⚙️ 技術的原理

### 単一サーバーの問題

```
クライアントA ──┐
クライアントB ──┼──► サーバー1
クライアントC ──┘

問題:
- すべてのクライアントが同じサーバーに接続
- サーバーのCPU・メモリに上限がある
- サーバーダウン時に全クライアントが切断
```

### 複数サーバー（Adapterなし）の問題

```
クライアントA ───► サーバー1
クライアントB ───► サーバー2

クライアントAが送信したメッセージは
クライアントBに届かない！
（異なるサーバーのため）
```

### Redis Adapterによる解決

```
クライアントA ───► サーバー1 ──┐
                            │
                        Redis ◄─► Pub/Sub
                            │
クライアントB ───► サーバー2 ──┘

1. クライアントAがメッセージを送信
2. サーバー1がRedisにPublish
3. Redis が全サーバーにBroadcast
4. サーバー2がメッセージを受信
5. クライアントBにメッセージを配信
```

### Sticky Session

```
ロードバランサー
    │
    ├─► サーバー1 (クライアントA, C)
    └─► サーバー2 (クライアントB, D)

同じクライアントは常に同じサーバーに接続
（セッションIDやIPアドレスでルーティング）

必要な理由:
- HTTP Long Polling使用時
- 接続確立時の複数リクエスト
```

---

## 💼 ユースケース

### 適している場面

1. **大規模チャット**: 数万〜数百万ユーザー
2. **リアルタイムゲーム**: 多数の同時接続
3. **ライブ配信**: 視聴者が多い配信
4. **IoTプラットフォーム**: 大量のデバイス接続
5. **グローバルサービス**: 世界中のユーザー

### 単一サーバーで十分な場合

1. **小規模アプリ**: 数百〜数千ユーザー
2. **プロトタイプ**: 開発初期段階
3. **社内ツール**: 限定されたユーザー数

---

## 💻 実装例

### ステップ1: プロジェクトのセットアップ

```bash
mkdir socket-scaling
cd socket-scaling
npm init -y
npm install express socket.io @socket.io/redis-adapter redis
```

### ステップ2: サーバー側の実装

`server.js`:

```javascript
const express = require('express');
const { createServer } = require('http');
const { Server } = require('socket.io');
const { createAdapter } = require('@socket.io/redis-adapter');
const { createClient } = require('redis');

// 環境変数からポートとサーバーIDを取得
const PORT = process.env.PORT || 3000;
const SERVER_ID = process.env.SERVER_ID || `server-${PORT}`;
const REDIS_URL = process.env.REDIS_URL || 'redis://localhost:6379';

const app = express();
const httpServer = createServer(app);
const io = new Server(httpServer, {
  cors: {
    origin: '*'
  }
});

app.use(express.static('public'));

// サーバー統計
const stats = {
  serverId: SERVER_ID,
  connections: 0,
  messages: 0,
  startTime: new Date()
};

// ===== Redis Adapterのセットアップ =====

async function setupRedisAdapter() {
  try {
    // Redisクライアントを2つ作成（Pub/Sub用）
    const pubClient = createClient({ url: REDIS_URL });
    const subClient = pubClient.duplicate();

    // エラーハンドリング
    pubClient.on('error', (err) => console.error('Redis Pub Client Error:', err));
    subClient.on('error', (err) => console.error('Redis Sub Client Error:', err));

    // 接続
    await pubClient.connect();
    await subClient.connect();

    console.log(`✅ Redis接続成功: ${REDIS_URL}`);

    // Socket.IOにRedis Adapterを設定
    io.adapter(createAdapter(pubClient, subClient));

    console.log(`✅ Redis Adapter設定完了`);

  } catch (error) {
    console.error('❌ Redis接続失敗:', error.message);
    console.log('⚠️ In-Memory Adapterで起動します（単一サーバーモード）');
  }
}

setupRedisAdapter();

// ===== Socket.IO接続処理 =====

io.on('connection', (socket) => {
  stats.connections++;
  console.log(`[${SERVER_ID}] ✅ 接続: ${socket.id} (合計: ${stats.connections})`);

  // クライアントにサーバー情報を送信
  socket.emit('serverInfo', {
    serverId: SERVER_ID,
    socketId: socket.id,
    timestamp: new Date().toISOString()
  });

  // ===== メッセージ処理 =====

  socket.on('message', (data) => {
    stats.messages++;
    console.log(`[${SERVER_ID}] 📨 メッセージ: ${socket.id} - ${data.text}`);

    // 送信元を除く全クライアントにブロードキャスト
    // Redis Adapterにより、他のサーバーのクライアントにも配信される
    socket.broadcast.emit('message', {
      from: socket.id,
      serverId: SERVER_ID,
      text: data.text,
      timestamp: new Date().toISOString()
    });

    // 送信元にACKを返す
    socket.emit('messageAck', {
      serverId: SERVER_ID,
      received: true,
      timestamp: new Date().toISOString()
    });
  });

  // ===== ルーム操作 =====

  socket.on('joinRoom', (room) => {
    socket.join(room);
    console.log(`[${SERVER_ID}] 🚪 ${socket.id} がルーム ${room} に参加`);

    // ルーム内の他のメンバーに通知
    // Redis Adapterにより、他のサーバーのルームメンバーにも配信
    socket.to(room).emit('userJoined', {
      socketId: socket.id,
      serverId: SERVER_ID,
      room: room,
      timestamp: new Date().toISOString()
    });

    socket.emit('roomJoined', { room: room });
  });

  socket.on('roomMessage', (data) => {
    console.log(`[${SERVER_ID}] 💬 ルームメッセージ: [${data.room}] ${data.text}`);

    // ルーム内にブロードキャスト
    socket.to(data.room).emit('roomMessage', {
      from: socket.id,
      serverId: SERVER_ID,
      room: data.room,
      text: data.text,
      timestamp: new Date().toISOString()
    });
  });

  // ===== 統計情報 =====

  socket.on('getStats', async () => {
    // 全サーバーの接続数を取得
    const sockets = await io.fetchSockets();

    socket.emit('stats', {
      serverId: SERVER_ID,
      localConnections: stats.connections,
      totalConnections: sockets.length,
      messages: stats.messages,
      uptime: Math.floor((Date.now() - stats.startTime) / 1000)
    });
  });

  // ===== 切断処理 =====

  socket.on('disconnect', () => {
    stats.connections--;
    console.log(`[${SERVER_ID}] ❌ 切断: ${socket.id} (残り: ${stats.connections})`);
  });
});

// ===== サーバー起動 =====

httpServer.listen(PORT, () => {
  console.log(`\n🚀 Socket.IOサーバー起動`);
  console.log(`   サーバーID: ${SERVER_ID}`);
  console.log(`   ポート: ${PORT}`);
  console.log(`   URL: http://localhost:${PORT}`);
  console.log(`   Redis: ${REDIS_URL}\n`);
});

// ===== API エンドポイント =====

app.get('/api/stats', async (req, res) => {
  const sockets = await io.fetchSockets();

  res.json({
    serverId: SERVER_ID,
    localConnections: stats.connections,
    totalConnections: sockets.length,
    messages: stats.messages,
    uptime: Math.floor((Date.now() - stats.startTime) / 1000)
  });
});

app.get('/health', (req, res) => {
  res.json({
    status: 'healthy',
    serverId: SERVER_ID,
    uptime: Math.floor((Date.now() - stats.startTime) / 1000)
  });
});

// ===== グレースフルシャットダウン =====

process.on('SIGTERM', async () => {
  console.log(`\n⚠️ ${SERVER_ID} シャットダウン開始...`);

  httpServer.close(async () => {
    console.log(`✅ ${SERVER_ID} HTTPサーバー停止`);

    // すべてのSocket.IO接続を閉じる
    io.close(() => {
      console.log(`✅ ${SERVER_ID} Socket.IO停止`);
      process.exit(0);
    });
  });

  // タイムアウト（30秒）
  setTimeout(() => {
    console.error(`❌ ${SERVER_ID} 強制終了（タイムアウト）`);
    process.exit(1);
  }, 30000);
});
```

### ステップ3: Docker Compose設定

`docker-compose.yml`:

```yaml
version: '3.8'

services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data

  server1:
    build: .
    environment:
      - PORT=3001
      - SERVER_ID=server-1
      - REDIS_URL=redis://redis:6379
    ports:
      - "3001:3001"
    depends_on:
      - redis

  server2:
    build: .
    environment:
      - PORT=3002
      - SERVER_ID=server-2
      - REDIS_URL=redis://redis:6379
    ports:
      - "3002:3002"
    depends_on:
      - redis

  server3:
    build: .
    environment:
      - PORT=3003
      - SERVER_ID=server-3
      - REDIS_URL=redis://redis:6379
    ports:
      - "3003:3003"
    depends_on:
      - redis

  nginx:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - server1
      - server2
      - server3

volumes:
  redis-data:
```

### ステップ4: Nginx設定（ロードバランサー）

`nginx.conf`:

```nginx
events {
    worker_connections 1024;
}

http {
    # アップストリーム定義（複数のSocket.IOサーバー）
    upstream socketio_backend {
        # IPハッシュによるSticky Session
        ip_hash;

        server server1:3001;
        server server2:3002;
        server server3:3003;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://socketio_backend;
            proxy_http_version 1.1;

            # WebSocketサポート
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";

            # その他のヘッダー
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # タイムアウト設定
            proxy_connect_timeout 60s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;
        }
    }
}
```

### ステップ5: Dockerfile

`Dockerfile`:

```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 3000

CMD ["node", "server.js"]
```

### ステップ6: クライアント側の実装

`public/index.html`:

```html
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Socket.IO - スケーリング</title>
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
    .grid {
      display: grid;
      grid-template-columns: 1fr 2fr;
      gap: 20px;
    }
    .card {
      background-color: white;
      border-radius: 8px;
      padding: 20px;
      box-shadow: 0 2px 5px rgba(0,0,0,0.1);
    }
    .card h2 {
      color: #666;
      font-size: 18px;
      margin-bottom: 15px;
      border-bottom: 2px solid #eee;
      padding-bottom: 10px;
    }
    .server-info {
      background-color: #e3f2fd;
      padding: 15px;
      border-radius: 5px;
      margin-bottom: 15px;
    }
    .server-id {
      font-size: 20px;
      font-weight: bold;
      color: #1976d2;
    }
    button {
      padding: 12px 20px;
      background-color: #4CAF50;
      color: white;
      border: none;
      border-radius: 5px;
      cursor: pointer;
      font-size: 14px;
      margin: 5px;
      width: calc(50% - 10px);
    }
    button:hover {
      background-color: #45a049;
    }
    input[type="text"] {
      width: 100%;
      padding: 10px;
      margin-bottom: 10px;
      border: 1px solid #ddd;
      border-radius: 5px;
      font-size: 14px;
    }
    .log {
      background-color: #f8f9fa;
      border: 1px solid #ddd;
      border-radius: 5px;
      padding: 10px;
      height: 500px;
      overflow-y: auto;
      font-family: monospace;
      font-size: 12px;
    }
    .log-entry {
      margin: 3px 0;
      padding: 5px;
      border-radius: 3px;
      border-left: 3px solid #4CAF50;
      background-color: white;
    }
  </style>
</head>
<body>
  <div class="container">
    <h1>⚡ Socket.IO - スケーリング（複数サーバー）</h1>

    <div class="grid">
      <div>
        <div class="card">
          <h2>📡 サーバー情報</h2>
          <div class="server-info">
            <div>接続先サーバー:</div>
            <div class="server-id" id="serverId">-</div>
            <div style="margin-top: 10px; font-size: 11px;">
              Socket ID: <span id="socketId" style="word-break: break-all;">-</span>
            </div>
          </div>

          <button onclick="connect()">接続</button>
          <button onclick="disconnect()">切断</button>
          <button onclick="getStats()">統計取得</button>
        </div>

        <div class="card" style="margin-top: 20px;">
          <h2>💬 メッセージ送信</h2>
          <input type="text" id="messageInput" placeholder="メッセージ">
          <button onclick="sendMessage()" style="width: 100%;">送信</button>
        </div>

        <div class="card" style="margin-top: 20px;">
          <h2>🚪 ルーム</h2>
          <input type="text" id="roomInput" placeholder="ルーム名" value="lobby">
          <button onclick="joinRoom()" style="width: 100%;">参加</button>
          <input type="text" id="roomMessageInput" placeholder="ルームメッセージ" style="margin-top: 10px;">
          <button onclick="sendRoomMessage()" style="width: 100%;">ルーム送信</button>
        </div>
      </div>

      <div class="card">
        <h2>📋 イベントログ</h2>
        <div id="log" class="log"></div>
        <button onclick="clearLog()" style="width: 100%; margin-top: 10px;">
          ログクリア
        </button>
      </div>
    </div>
  </div>

  <script src="/socket.io/socket.io.js"></script>
  <script>
    let socket = null;

    function addLog(message) {
      const log = document.getElementById('log');
      const entry = document.createElement('div');
      entry.className = 'log-entry';
      const time = new Date().toLocaleTimeString();
      entry.textContent = `[${time}] ${message}`;
      log.appendChild(entry);
      log.scrollTop = log.scrollHeight;
    }

    function clearLog() {
      document.getElementById('log').innerHTML = '';
    }

    function connect() {
      if (socket) socket.disconnect();

      // ロードバランサー経由で接続（ポート8080）
      // または直接サーバーに接続（ポート3001, 3002, 3003）
      socket = io('/');

      socket.on('connect', () => {
        addLog('✅ 接続成功');
      });

      socket.on('serverInfo', (data) => {
        document.getElementById('serverId').textContent = data.serverId;
        document.getElementById('socketId').textContent = data.socketId;
        addLog(`📡 サーバー情報: ${data.serverId} (${data.socketId})`);
      });

      socket.on('message', (data) => {
        addLog(`💬 [${data.serverId}] ${data.from}: ${data.text}`);
      });

      socket.on('messageAck', (data) => {
        addLog(`✅ ACK from ${data.serverId}`);
      });

      socket.on('roomJoined', (data) => {
        addLog(`🚪 ルーム参加: ${data.room}`);
      });

      socket.on('userJoined', (data) => {
        addLog(`👤 [${data.serverId}] ${data.socketId} がルーム ${data.room} に参加`);
      });

      socket.on('roomMessage', (data) => {
        addLog(`💬 [${data.room}] [${data.serverId}] ${data.from}: ${data.text}`);
      });

      socket.on('stats', (data) => {
        addLog(`📊 統計: ${data.serverId} - ローカル=${data.localConnections}, 合計=${data.totalConnections}, メッセージ=${data.messages}, 稼働時間=${data.uptime}秒`);
      });

      socket.on('disconnect', () => {
        addLog('❌ 切断');
      });
    }

    function disconnect() {
      if (socket) {
        socket.disconnect();
        socket = null;
      }
    }

    function sendMessage() {
      if (!socket || !socket.connected) {
        alert('接続してください');
        return;
      }

      const text = document.getElementById('messageInput').value.trim();
      if (!text) return;

      socket.emit('message', { text: text });
      addLog(`📤 送信: ${text}`);
      document.getElementById('messageInput').value = '';
    }

    function joinRoom() {
      if (!socket || !socket.connected) {
        alert('接続してください');
        return;
      }

      const room = document.getElementById('roomInput').value.trim();
      if (!room) return;

      socket.emit('joinRoom', room);
    }

    function sendRoomMessage() {
      if (!socket || !socket.connected) {
        alert('接続してください');
        return;
      }

      const room = document.getElementById('roomInput').value.trim();
      const text = document.getElementById('roomMessageInput').value.trim();

      if (!room || !text) return;

      socket.emit('roomMessage', { room: room, text: text });
      addLog(`📤 [${room}] 送信: ${text}`);
      document.getElementById('roomMessageInput').value = '';
    }

    function getStats() {
      if (!socket || !socket.connected) {
        alert('接続してください');
        return;
      }

      socket.emit('getStats');
    }

    // 自動接続
    window.addEventListener('load', () => {
      connect();
    });
  </script>
</body>
</html>
```

---

## 🔍 スケーリングのベストプラクティス

### 1. Sticky Sessionの設定

```nginx
# Nginxでのip_hash
upstream backend {
    ip_hash;
    server server1:3001;
    server server2:3002;
}
```

### 2. ヘルスチェック

```javascript
app.get('/health', (req, res) => {
  res.json({ status: 'healthy' });
});
```

### 3. グレースフルシャットダウン

```javascript
process.on('SIGTERM', () => {
  server.close(() => {
    io.close();
    process.exit(0);
  });
});
```

---

## 🧪 動作確認

### ローカルでのテスト

```bash
# Redisを起動
docker run -p 6379:6379 redis

# 複数のサーバーを起動
PORT=3001 SERVER_ID=server-1 node server.js &
PORT=3002 SERVER_ID=server-2 node server.js &
PORT=3003 SERVER_ID=server-3 node server.js &

# 複数のブラウザで異なるポートに接続
# http://localhost:3001
# http://localhost:3002
# http://localhost:3003

# メッセージが全サーバーのクライアントに配信されることを確認
```

---

## 📝 練習課題

### 初級
1. **サーバーモニタリング**: 各サーバーの接続数をリアルタイム表示
2. **ヘルスチェック**: 定期的にサーバーの健全性を確認

### 中級
3. **動的スケーリング**: 負荷に応じてサーバー数を自動増減
4. **地理的ルーティング**: ユーザーの位置に応じて最適なサーバーに接続

### 上級
5. **Kubernetes対応**: K8sでのオートスケーリング
6. **マルチリージョン**: 複数のデータセンターでの冗長化

---

## 🎓 まとめ

おめでとうございます！Socket.IOの全10レッスンを完了しました！

これまでに学んだこと：
1. ✅ Socket.IOの全体像と技術的原理
2. ✅ 基本的な接続とイベントの送受信
3. ✅ ブロードキャストとルーム
4. ✅ ネームスペースとミドルウェア
5. ✅ エラーハンドリングと再接続
6. ✅ スケーリングと本番運用

次のステップ：
- 実際のプロジェクトで実装してみる
- パフォーマンステストを実施する
- セキュリティ対策を強化する
- モニタリングとログ分析を導入する

---

## 📚 参考リソース

- [Socket.IO - Using multiple nodes](https://socket.io/docs/v4/using-multiple-nodes/)
- [Socket.IO - Redis Adapter](https://socket.io/docs/v4/redis-adapter/)
- [Nginx - Load Balancing](https://nginx.org/en/docs/http/load_balancing.html)
