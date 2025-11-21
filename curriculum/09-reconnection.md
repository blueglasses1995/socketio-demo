# Lesson 9: 再接続とバッファリング

## 🎯 学習目標

- 自動再接続の仕組みを理解する
- 再接続の設定をカスタマイズする
- バッファリングを活用してメッセージを保持する
- 切断と再接続のライフサイクルを管理する

---

## 📋 メリット

✅ **高い可用性**: 一時的な切断からの自動復旧
✅ **ユーザー体験**: シームレスな接続維持
✅ **メッセージ保証**: 切断中のメッセージもバッファリング
✅ **ネットワーク耐性**: 不安定なネットワークでも動作
✅ **自動化**: 手動での再接続処理が不要

---

## ⚠️ デメリット

❌ **無限ループ**: 設定ミスで無限再接続の可能性
❌ **リソース消費**: 再接続試行でCPU・ネットワーク消費
❌ **バッファ制限**: 無制限にバッファすると メモリオーバーフロー
❌ **状態の不整合**: 再接続後の状態同期が必要

---

## ⚙️ 技術的原理

### 再接続のフロー

```
接続中
    ↓
切断検出（ping timeout / network error等）
    ↓
reconnection_attempt イベント発火
    ↓
指定された遅延時間待機（exponential backoff）
    ↓
再接続試行
    ↓
成功 → reconnect イベント発火
失敗 → 次の試行へ（reconnection_attemptsまで）
```

### Exponential Backoff（指数バックオフ）

```
試行1: 1秒待機
試行2: 2秒待機
試行3: 4秒待機
試行4: 8秒待機
試行5: 16秒待機
...

最大遅延時間に達したら一定間隔で試行
```

### バッファリングの仕組み

```
接続中:
  socket.emit('event', data)  → 即座に送信

切断中:
  socket.emit('event', data)  → バッファに保存

再接続後:
  バッファ内のイベントを順番に送信
```

---

## 💼 ユースケース

### 適している場面

1. **モバイルアプリ**: ネットワークの切り替え（WiFi ↔ 4G）
2. **スリープ復帰**: デバイスがスリープから復帰した時
3. **サーバーメンテナンス**: 短時間のサーバー再起動
4. **不安定なネットワーク**: 弱い電波環境での利用
5. **ロングポーリング**: WebSocketが使えない環境

### 注意が必要な場面

1. **リアルタイムゲーム**: 古いデータが送信されると問題
2. **高頻度送信**: バッファが大きくなりすぎる
3. **セキュリティ**: 再接続時の認証更新が必要

---

## 💻 実装例

### ステップ1: プロジェクトのセットアップ

```bash
mkdir socket-reconnection
cd socket-reconnection
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
const io = new Server(httpServer, {
  // Ping/Pongの設定
  pingTimeout: 10000,  // 10秒以内にPong応答がなければ切断
  pingInterval: 5000   // 5秒ごとにPingを送信
});

app.use(express.static('public'));

// 接続中のクライアントを追跡
const connectedClients = new Map();

io.on('connection', (socket) => {
  const connectTime = new Date();
  console.log(`✅ 接続: ${socket.id} (${connectTime.toLocaleTimeString()})`);

  // クライアント情報を保存
  connectedClients.set(socket.id, {
    id: socket.id,
    connectedAt: connectTime,
    reconnectCount: socket.data.reconnectCount || 0,
    messagesReceived: 0
  });

  // ウェルカムメッセージ
  socket.emit('welcome', {
    socketId: socket.id,
    serverTime: connectTime.toISOString(),
    reconnectCount: socket.data.reconnectCount || 0
  });

  // ===== メッセージ受信 =====

  socket.on('message', (data) => {
    const client = connectedClients.get(socket.id);
    if (client) {
      client.messagesReceived++;
    }

    console.log(`📨 メッセージ: ${socket.id} - ${data.text} (${data.timestamp})`);

    // エコーバック
    socket.emit('messageAck', {
      receivedAt: new Date().toISOString(),
      originalTimestamp: data.timestamp,
      text: data.text
    });

    // 他のクライアントにブロードキャスト
    socket.broadcast.emit('message', {
      from: socket.id,
      text: data.text,
      timestamp: new Date().toISOString()
    });
  });

  // ===== 統計情報の取得 =====

  socket.on('getStats', () => {
    const client = connectedClients.get(socket.id);
    socket.emit('stats', {
      socketId: socket.id,
      connectedAt: client.connectedAt,
      reconnectCount: client.reconnectCount,
      messagesReceived: client.messagesReceived,
      totalConnected: connectedClients.size
    });
  });

  // ===== 強制切断（テスト用） =====

  socket.on('forceDisconnect', () => {
    console.log(`⚠️ 強制切断: ${socket.id}`);
    socket.disconnect(true);
  });

  // ===== 切断処理 =====

  socket.on('disconnect', (reason) => {
    console.log(`❌ 切断: ${socket.id} - 理由: ${reason}`);

    // 再接続の可能性があるためすぐには削除しない
    setTimeout(() => {
      if (connectedClients.has(socket.id)) {
        // まだ再接続していない場合のみ削除
        const sockets = Array.from(io.sockets.sockets.keys());
        if (!sockets.includes(socket.id)) {
          connectedClients.delete(socket.id);
          console.log(`🗑️ クライアント情報を削除: ${socket.id}`);
        }
      }
    }, 60000); // 60秒後に削除
  });

  // ===== 再接続検出 =====

  socket.on('reconnect_info', (data) => {
    console.log(`🔄 再接続情報: ${socket.id} - 試行回数: ${data.attemptNumber}`);

    // 再接続カウントを増やす
    const client = connectedClients.get(socket.id);
    if (client) {
      client.reconnectCount++;
      socket.data.reconnectCount = client.reconnectCount;
    }
  });

  // ===== Heartbeat（生存確認） =====

  socket.on('heartbeat', () => {
    socket.emit('heartbeat_ack', {
      serverTime: new Date().toISOString()
    });
  });
});

// ===== サーバー統計API =====

app.get('/api/stats', (req, res) => {
  const stats = {
    connectedClients: connectedClients.size,
    clients: Array.from(connectedClients.values())
  };
  res.json(stats);
});

// ===== サーバー再起動シミュレーション（デモ用） =====

app.post('/api/restart', (req, res) => {
  console.log('⚠️ サーバー再起動シミュレーション...');
  res.json({ message: 'すべてのクライアントを切断します' });

  // すべてのクライアントを切断
  io.disconnectSockets();

  setTimeout(() => {
    console.log('✅ サーバー再起動完了（クライアントは自動再接続するはず）');
  }, 1000);
});

const PORT = 3000;
httpServer.listen(PORT, () => {
  console.log(`🚀 サーバー起動: http://localhost:${PORT}`);
  console.log(`設定:`);
  console.log(`  - pingTimeout: 10秒`);
  console.log(`  - pingInterval: 5秒`);
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
  <title>Socket.IO - 再接続とバッファリング</title>
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
      grid-template-columns: 1fr 1fr;
      gap: 20px;
      margin-bottom: 20px;
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
    .status {
      padding: 15px;
      border-radius: 5px;
      margin-bottom: 15px;
      font-weight: bold;
      text-align: center;
    }
    .status.connected {
      background-color: #d4edda;
      color: #155724;
    }
    .status.disconnected {
      background-color: #f8d7da;
      color: #721c24;
    }
    .status.reconnecting {
      background-color: #fff3cd;
      color: #856404;
    }
    .info-grid {
      display: grid;
      grid-template-columns: 1fr 1fr;
      gap: 10px;
      margin-bottom: 15px;
    }
    .info-item {
      background-color: #f8f9fa;
      padding: 10px;
      border-radius: 5px;
    }
    .info-label {
      font-size: 12px;
      color: #666;
    }
    .info-value {
      font-size: 18px;
      font-weight: bold;
      color: #333;
      margin-top: 5px;
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
    button.danger {
      background-color: #f44336;
    }
    button.danger:hover {
      background-color: #da190b;
    }
    button.warning {
      background-color: #ff9800;
    }
    button.warning:hover {
      background-color: #f57c00;
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
      height: 400px;
      overflow-y: auto;
      font-family: monospace;
      font-size: 12px;
    }
    .log-entry {
      margin: 3px 0;
      padding: 5px;
      border-radius: 3px;
    }
    .log-entry.info {
      background-color: #e3f2fd;
      border-left: 3px solid #2196F3;
    }
    .log-entry.success {
      background-color: #e8f5e9;
      border-left: 3px solid #4CAF50;
    }
    .log-entry.warning {
      background-color: #fff3e0;
      border-left: 3px solid #ff9800;
    }
    .log-entry.error {
      background-color: #ffebee;
      border-left: 3px solid #f44336;
    }
    .config-group {
      background-color: #f8f9fa;
      padding: 15px;
      border-radius: 5px;
      margin-bottom: 15px;
    }
    .config-item {
      margin: 10px 0;
    }
    .config-item label {
      display: block;
      font-size: 12px;
      color: #666;
      margin-bottom: 5px;
    }
    .config-item input[type="number"] {
      width: 100%;
      padding: 8px;
      border: 1px solid #ddd;
      border-radius: 4px;
    }
  </style>
</head>
<body>
  <div class="container">
    <h1>🔄 Socket.IO - 再接続とバッファリング</h1>

    <div class="grid">
      <!-- 接続状態 -->
      <div class="card">
        <h2>📡 接続状態</h2>

        <div id="statusDisplay" class="status disconnected">
          ⚫ 未接続
        </div>

        <div class="info-grid">
          <div class="info-item">
            <div class="info-label">Socket ID</div>
            <div class="info-value" id="socketId" style="font-size: 11px; word-break: break-all;">-</div>
          </div>
          <div class="info-item">
            <div class="info-label">再接続回数</div>
            <div class="info-value" id="reconnectCount">0</div>
          </div>
          <div class="info-item">
            <div class="info-label">メッセージ送信</div>
            <div class="info-value" id="messagesSent">0</div>
          </div>
          <div class="info-item">
            <div class="info-label">メッセージ受信</div>
            <div class="info-value" id="messagesReceived">0</div>
          </div>
        </div>

        <div>
          <button onclick="connect()">接続</button>
          <button onclick="disconnect()" class="danger">切断</button>
          <button onclick="forceDisconnect()" class="warning">強制切断</button>
          <button onclick="getStats()">統計取得</button>
        </div>
      </div>

      <!-- 再接続設定 -->
      <div class="card">
        <h2>⚙️ 再接続設定</h2>

        <div class="config-group">
          <div class="config-item">
            <label>再接続を有効化</label>
            <input type="checkbox" id="reconnection" checked>
          </div>
          <div class="config-item">
            <label>再接続試行回数（Infinityで無限）</label>
            <input type="number" id="reconnectionAttempts" value="5" min="1">
          </div>
          <div class="config-item">
            <label>初回再接続遅延（ミリ秒）</label>
            <input type="number" id="reconnectionDelay" value="1000" min="100">
          </div>
          <div class="config-item">
            <label>最大再接続遅延（ミリ秒）</label>
            <input type="number" id="reconnectionDelayMax" value="5000" min="1000">
          </div>
          <div class="config-item">
            <label>タイムアウト（ミリ秒）</label>
            <input type="number" id="timeout" value="20000" min="1000">
          </div>
        </div>

        <button onclick="applyConfig()" style="width: 100%;">設定を適用して再接続</button>
      </div>

      <!-- メッセージ送信 -->
      <div class="card">
        <h2>💬 メッセージ送信</h2>

        <input type="text" id="messageInput" placeholder="メッセージを入力...">
        <div>
          <button onclick="sendMessage()" style="width: 100%;">送信</button>
        </div>

        <div style="margin-top: 15px;">
          <button onclick="sendBurst(5)">5件連続送信</button>
          <button onclick="sendBurst(10)">10件連続送信</button>
        </div>

        <div style="margin-top: 10px; padding: 10px; background-color: #e3f2fd; border-radius: 5px; font-size: 13px;">
          💡 切断中に送信したメッセージは再接続後に自動送信されます
        </div>
      </div>

      <!-- イベントログ -->
      <div class="card">
        <h2>📋 イベントログ</h2>
        <div id="log" class="log"></div>
        <button onclick="clearLog()" class="danger" style="width: 100%; margin-top: 10px;">
          ログクリア
        </button>
      </div>
    </div>
  </div>

  <script src="/socket.io/socket.io.js"></script>
  <script>
    let socket = null;
    let messagesSent = 0;
    let messagesReceived = 0;
    let reconnectCount = 0;

    function addLog(message, type = 'info') {
      const log = document.getElementById('log');
      const entry = document.createElement('div');
      entry.className = `log-entry ${type}`;
      const time = new Date().toLocaleTimeString();
      entry.textContent = `[${time}] ${message}`;
      log.appendChild(entry);
      log.scrollTop = log.scrollHeight;
    }

    function clearLog() {
      document.getElementById('log').innerHTML = '';
    }

    function updateStatus(status, text) {
      const statusDiv = document.getElementById('statusDisplay');
      statusDiv.className = `status ${status}`;
      statusDiv.textContent = text;
    }

    function updateStats() {
      document.getElementById('messagesSent').textContent = messagesSent;
      document.getElementById('messagesReceived').textContent = messagesReceived;
      document.getElementById('reconnectCount').textContent = reconnectCount;
    }

    function getConfig() {
      const attempts = document.getElementById('reconnectionAttempts').value;
      return {
        reconnection: document.getElementById('reconnection').checked,
        reconnectionAttempts: attempts === 'Infinity' ? Infinity : parseInt(attempts),
        reconnectionDelay: parseInt(document.getElementById('reconnectionDelay').value),
        reconnectionDelayMax: parseInt(document.getElementById('reconnectionDelayMax').value),
        timeout: parseInt(document.getElementById('timeout').value)
      };
    }

    function connect() {
      if (socket && socket.connected) {
        addLog('⚠️ すでに接続されています', 'warning');
        return;
      }

      const config = getConfig();
      addLog(`🔌 接続中... (設定: ${JSON.stringify(config)})`, 'info');

      socket = io('/', config);

      // ===== 接続イベント =====

      socket.on('connect', () => {
        addLog('✅ 接続成功', 'success');
        updateStatus('connected', '🟢 接続中');
        document.getElementById('socketId').textContent = socket.id;
      });

      socket.on('welcome', (data) => {
        addLog(`👋 ウェルカムメッセージ: Socket ID=${data.socketId}, 再接続=${data.reconnectCount}回`, 'info');
        reconnectCount = data.reconnectCount;
        updateStats();
      });

      // ===== 切断イベント =====

      socket.on('disconnect', (reason) => {
        addLog(`❌ 切断: ${reason}`, 'error');
        updateStatus('disconnected', '⚫ 切断');
      });

      socket.on('connect_error', (error) => {
        addLog(`⚠️ 接続エラー: ${error.message}`, 'error');
      });

      // ===== 再接続イベント =====

      socket.io.on('reconnect_attempt', (attemptNumber) => {
        addLog(`🔄 再接続試行 #${attemptNumber}`, 'warning');
        updateStatus('reconnecting', `🔄 再接続中 (試行 ${attemptNumber})`);
      });

      socket.io.on('reconnect', (attemptNumber) => {
        addLog(`✅ 再接続成功 (試行回数: ${attemptNumber})`, 'success');
        updateStatus('connected', '🟢 接続中 (再接続)');
        reconnectCount++;
        updateStats();

        // 再接続情報をサーバーに送信
        socket.emit('reconnect_info', { attemptNumber: attemptNumber });
      });

      socket.io.on('reconnect_error', (error) => {
        addLog(`⚠️ 再接続エラー: ${error.message}`, 'error');
      });

      socket.io.on('reconnect_failed', () => {
        addLog(`❌ 再接続失敗: 最大試行回数に達しました`, 'error');
        updateStatus('disconnected', '⚫ 再接続失敗');
      });

      // ===== メッセージイベント =====

      socket.on('messageAck', (data) => {
        messagesReceived++;
        updateStats();
        addLog(`📨 ACK受信: "${data.text}" (送信: ${data.originalTimestamp}, 受信: ${data.receivedAt})`, 'success');
      });

      socket.on('message', (data) => {
        messagesReceived++;
        updateStats();
        addLog(`💬 メッセージ受信: ${data.from} - ${data.text}`, 'info');
      });

      socket.on('stats', (data) => {
        addLog(`📊 統計: 再接続=${data.reconnectCount}回, メッセージ受信=${data.messagesReceived}件, 合計接続=${data.totalConnected}`, 'info');
      });

      // ===== Ping/Pongイベント（デバッグ用） =====

      socket.io.on('ping', () => {
        addLog('📡 Ping送信', 'info');
      });

      socket.io.on('pong', (latency) => {
        addLog(`📡 Pong受信 (遅延: ${latency}ms)`, 'info');
      });
    }

    function disconnect() {
      if (!socket) {
        addLog('⚠️ 接続されていません', 'warning');
        return;
      }

      addLog('🔌 切断中...', 'info');
      socket.disconnect();
    }

    function forceDisconnect() {
      if (!socket || !socket.connected) {
        addLog('⚠️ 接続してください', 'warning');
        return;
      }

      addLog('💥 サーバーに強制切断を要求', 'warning');
      socket.emit('forceDisconnect');
    }

    function sendMessage() {
      if (!socket) {
        addLog('⚠️ 接続してください', 'warning');
        return;
      }

      const input = document.getElementById('messageInput');
      const text = input.value.trim();

      if (!text) return;

      const message = {
        text: text,
        timestamp: new Date().toISOString()
      };

      addLog(`📤 メッセージ送信: "${text}" (${socket.connected ? '接続中' : 'バッファリング'})`, 'info');

      socket.emit('message', message);
      messagesSent++;
      updateStats();

      input.value = '';
    }

    function sendBurst(count) {
      for (let i = 1; i <= count; i++) {
        setTimeout(() => {
          const message = {
            text: `バーストメッセージ #${i}`,
            timestamp: new Date().toISOString()
          };

          socket.emit('message', message);
          messagesSent++;
          updateStats();
        }, i * 100);
      }

      addLog(`📤 ${count}件のメッセージを送信予約`, 'info');
    }

    function getStats() {
      if (!socket || !socket.connected) {
        addLog('⚠️ 接続してください', 'warning');
        return;
      }

      socket.emit('getStats');
    }

    function applyConfig() {
      if (socket) {
        socket.disconnect();
      }

      setTimeout(() => {
        connect();
      }, 500);
    }

    // Enterキーで送信
    document.getElementById('messageInput').addEventListener('keypress', (e) => {
      if (e.key === 'Enter') {
        sendMessage();
      }
    });

    // 自動接続
    window.addEventListener('load', () => {
      connect();
    });
  </script>
</body>
</html>
```

---

## 🔍 再接続の設定オプション

### クライアント側の設定

```javascript
const socket = io('/', {
  // 再接続を有効化
  reconnection: true,

  // 再接続試行回数（Infinityで無限）
  reconnectionAttempts: 5,

  // 初回の再接続遅延（ミリ秒）
  reconnectionDelay: 1000,

  // 最大再接続遅延（ミリ秒）
  reconnectionDelayMax: 5000,

  // 乱数化係数（0-1、デフォルト0.5）
  randomizationFactor: 0.5,

  // タイムアウト（ミリ秒）
  timeout: 20000
});
```

### サーバー側の設定

```javascript
const io = new Server(httpServer, {
  // Ping/Pongのタイムアウト
  pingTimeout: 10000,

  // Ping送信間隔
  pingInterval: 5000
});
```

---

## 🧪 動作確認

### テスト手順

1. サーバー起動: `node server.js`
2. ブラウザで `http://localhost:3000` を開く
3. 自動接続されることを確認
4. メッセージを送信
5. 「強制切断」をクリック
6. 自動的に再接続されることを確認
7. バッファリングされたメッセージが送信されることを確認

---

## 📝 練習課題

### 初級
1. **再接続カウンター**: 再接続回数を表示
2. **接続時間**: 接続してからの経過時間を表示

### 中級
3. **オフラインキュー**: 切断中のメッセージをローカルストレージに保存
4. **状態同期**: 再接続後にサーバーから最新状態を取得

### 上級
5. **スマート再接続**: ネットワーク状態に応じて再接続戦略を変更
6. **メッセージID**: すべてのメッセージにIDを付与して重複送信を防ぐ

---

## 💡 次のステップ

再接続とバッファリングをマスターしたら、最後に「スケーリング」を学んで、複数サーバーでの運用方法を学びましょう！

👉 [Lesson 10: スケーリング](./10-scaling.md)

---

## 📚 参考リソース

- [Socket.IO - Client initialization](https://socket.io/docs/v4/client-initialization/)
- [Socket.IO - Connection state recovery](https://socket.io/docs/v4/connection-state-recovery/)
