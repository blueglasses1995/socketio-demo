# Lesson 7: ミドルウェア

## 🎯 学習目標

- ミドルウェアの概念を理解する
- 認証ミドルウェアを実装する
- ロギングミドルウェアを実装する
- エラーハンドリングミドルウェアを実装する

---

## 📋 メリット

✅ **認証・認可**: 接続前にユーザーを検証できる
✅ **ロギング**: すべての接続をログに記録できる
✅ **前処理**: 接続前にデータを加工・検証できる
✅ **エラーハンドリング**: 不正な接続を拒否できる
✅ **再利用性**: 共通処理をモジュール化できる

---

## ⚠️ デメリット

❌ **パフォーマンス**: ミドルウェアが多いと接続が遅くなる
❌ **複雑性**: 多段のミドルウェアでデバッグが困難
❌ **順序依存**: ミドルウェアの実行順序に注意が必要
❌ **非同期処理**: async処理でエラーハンドリングが複雑

---

## ⚙️ 技術的原理

### ミドルウェアの実行フロー

```
クライアント接続要求
    ↓
ミドルウェア1 (認証チェック)
    ↓
ミドルウェア2 (ロギング)
    ↓
ミドルウェア3 (データ検証)
    ↓
next() 呼び出し → 接続成功
または
next(error) 呼び出し → 接続拒否
```

### 内部動作

```javascript
// ミドルウェアの内部処理
1. 接続リクエスト受信
2. ミドルウェアスタックを順番に実行
3. 各ミドルウェアで socket と next を渡す
4. next() が呼ばれたら次のミドルウェアへ
5. next(error) が呼ばれたら接続拒否
6. すべてのミドルウェアが成功したら 'connection' イベント発火
```

### ミドルウェアのスコープ

```javascript
// グローバルミドルウェア（すべての接続に適用）
io.use((socket, next) => { ... });

// ネームスペースミドルウェア（特定のネームスペースのみ）
const nsp = io.of('/admin');
nsp.use((socket, next) => { ... });
```

---

## 💼 ユースケース

### 適している場面

1. **JWT認証**: トークンベースの認証
2. **APIキー検証**: API接続の認証
3. **レート制限**: 接続頻度の制限
4. **IPホワイトリスト**: 特定のIPのみ許可
5. **ロギング**: 接続ログの記録
6. **A/Bテスト**: ユーザーグループの振り分け

### 注意が必要な場面

1. **重い処理**: データベースクエリ等の時間がかかる処理
2. **状態管理**: ミドルウェア間で状態を共有する場合
3. **非同期処理**: Promiseやasync/awaitを使う場合

---

## 💻 実装例

### ステップ1: プロジェクトのセットアップ

```bash
mkdir socket-middleware
cd socket-middleware
npm init -y
npm install express socket.io jsonwebtoken
```

### ステップ2: サーバー側の実装

`server.js`:

```javascript
const express = require('express');
const { createServer } = require('http');
const { Server } = require('socket.io');
const jwt = require('jsonwebtoken');

const app = express();
const httpServer = createServer(app);
const io = new Server(httpServer);

app.use(express.static('public'));

// JWT秘密鍵
const JWT_SECRET = 'your-secret-key';

// 接続ログを保存
const connectionLogs = [];

// レート制限用のマップ
const rateLimitMap = new Map();

// ===== ミドルウェア1: ロギング =====

io.use((socket, next) => {
  const log = {
    timestamp: new Date().toISOString(),
    socketId: socket.id,
    ip: socket.handshake.address,
    userAgent: socket.handshake.headers['user-agent']
  };

  connectionLogs.push(log);
  console.log('📝 接続ログ:', log);

  // socket.dataに情報を保存（後で使用可能）
  socket.data.connectedAt = log.timestamp;

  next();
});

// ===== ミドルウェア2: レート制限 =====

io.use((socket, next) => {
  const ip = socket.handshake.address;
  const now = Date.now();
  const limit = 5; // 5秒以内に1接続のみ許可
  const timeWindow = 5000;

  if (rateLimitMap.has(ip)) {
    const lastConnection = rateLimitMap.get(ip);
    const timeDiff = now - lastConnection;

    if (timeDiff < timeWindow) {
      const remainingTime = Math.ceil((timeWindow - timeDiff) / 1000);
      console.log(`⚠️ レート制限: ${ip} (${remainingTime}秒待機必要)`);
      return next(new Error(`接続が早すぎます。${remainingTime}秒後に再試行してください`));
    }
  }

  rateLimitMap.set(ip, now);
  next();
});

// ===== ミドルウェア3: JWT認証（オプショナル） =====

io.use((socket, next) => {
  const token = socket.handshake.auth.token;

  // トークンが提供されていない場合はゲストとして接続
  if (!token) {
    socket.data.user = {
      id: `guest_${socket.id}`,
      username: 'ゲスト',
      role: 'guest'
    };
    console.log('👤 ゲストユーザー:', socket.data.user);
    return next();
  }

  // トークンを検証
  try {
    const decoded = jwt.verify(token, JWT_SECRET);
    socket.data.user = decoded;
    console.log('✅ 認証成功:', decoded);
    next();
  } catch (err) {
    console.log('❌ 認証失敗:', err.message);
    next(new Error('認証に失敗しました'));
  }
});

// ===== ミドルウェア4: カスタムヘッダー検証 =====

io.use((socket, next) => {
  const apiKey = socket.handshake.headers['x-api-key'];

  // APIキーが必要な場合のチェック（デモでは任意）
  if (apiKey && apiKey !== 'valid-api-key') {
    console.log('❌ 無効なAPIキー:', apiKey);
    return next(new Error('無効なAPIキーです'));
  }

  if (apiKey) {
    socket.data.apiKeyValid = true;
    console.log('🔑 APIキー検証成功');
  }

  next();
});

// ===== 接続処理 =====

io.on('connection', (socket) => {
  console.log(`\n✅ 接続成功: ${socket.id}`);
  console.log(`   ユーザー: ${socket.data.user.username} (${socket.data.user.role})`);
  console.log(`   接続時刻: ${socket.data.connectedAt}`);

  // ユーザー情報を送信
  socket.emit('authenticated', {
    user: socket.data.user,
    connectedAt: socket.data.connectedAt
  });

  // ===== 管理者専用イベント =====

  socket.on('getConnectionLogs', () => {
    // 管理者のみログを取得可能
    if (socket.data.user.role !== 'admin') {
      socket.emit('error', {
        message: '権限がありません'
      });
      return;
    }

    socket.emit('connectionLogs', connectionLogs);
  });

  socket.on('disconnect', () => {
    console.log(`❌ 切断: ${socket.id} (${socket.data.user.username})`);
  });
});

// ===== 管理者用ネームスペース（厳格な認証） =====

const adminNamespace = io.of('/admin');

// 管理者ネームスペース専用ミドルウェア
adminNamespace.use((socket, next) => {
  const token = socket.handshake.auth.token;

  if (!token) {
    return next(new Error('認証トークンが必要です'));
  }

  try {
    const decoded = jwt.verify(token, JWT_SECRET);

    // 管理者権限をチェック
    if (decoded.role !== 'admin') {
      return next(new Error('管理者権限が必要です'));
    }

    socket.data.user = decoded;
    next();
  } catch (err) {
    next(new Error('認証に失敗しました'));
  }
});

adminNamespace.on('connection', (socket) => {
  console.log(`\n👑 管理者接続: ${socket.id}`);
  console.log(`   ユーザー: ${socket.data.user.username}`);

  socket.emit('welcome', {
    message: '管理者ダッシュボードへようこそ',
    user: socket.data.user
  });

  socket.on('getStats', async () => {
    const sockets = await io.fetchSockets();
    const adminSockets = await adminNamespace.fetchSockets();

    socket.emit('stats', {
      totalConnections: sockets.length,
      adminConnections: adminSockets.length,
      logs: connectionLogs.length
    });
  });
});

// ===== トークン生成エンドポイント（デモ用） =====

app.get('/api/token', (req, res) => {
  const role = req.query.role || 'user';
  const username = req.query.username || 'ユーザー';

  const token = jwt.sign(
    {
      id: Date.now(),
      username: username,
      role: role
    },
    JWT_SECRET,
    { expiresIn: '1h' }
  );

  res.json({ token, role, username });
});

app.get('/api/logs', (req, res) => {
  res.json(connectionLogs);
});

const PORT = 3000;
httpServer.listen(PORT, () => {
  console.log(`🚀 サーバー起動: http://localhost:${PORT}`);
  console.log(`\nトークン取得:`);
  console.log(`  - ゲスト: トークンなしで接続`);
  console.log(`  - ユーザー: http://localhost:${PORT}/api/token?role=user&username=太郎`);
  console.log(`  - 管理者: http://localhost:${PORT}/api/token?role=admin&username=管理者`);
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
  <title>Socket.IO - ミドルウェア</title>
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
    }
    h1 {
      color: #333;
      margin-bottom: 20px;
      border-bottom: 3px solid #4CAF50;
      padding-bottom: 10px;
    }
    .grid {
      display: grid;
      grid-template-columns: repeat(auto-fit, minmax(400px, 1fr));
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
    .user-info {
      background-color: #e3f2fd;
      padding: 15px;
      border-radius: 5px;
      margin-bottom: 15px;
    }
    .user-info.guest {
      background-color: #fff3e0;
    }
    .user-info.admin {
      background-color: #f3e5f5;
    }
    .status {
      display: inline-block;
      padding: 5px 12px;
      border-radius: 15px;
      font-size: 12px;
      font-weight: bold;
      margin-left: 10px;
    }
    .status.connected {
      background-color: #d4edda;
      color: #155724;
    }
    .status.disconnected {
      background-color: #f8d7da;
      color: #721c24;
    }
    .input-group {
      display: flex;
      gap: 10px;
      margin-bottom: 10px;
    }
    input {
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
    button.secondary {
      background-color: #2196F3;
    }
    button.secondary:hover {
      background-color: #0b7dda;
    }
    button.danger {
      background-color: #f44336;
    }
    button.danger:hover {
      background-color: #da190b;
    }
    .log {
      background-color: #f8f9fa;
      border: 1px solid #ddd;
      border-radius: 5px;
      padding: 10px;
      height: 250px;
      overflow-y: auto;
      font-family: monospace;
      font-size: 12px;
    }
    .log-entry {
      margin: 3px 0;
      padding: 5px;
      border-left: 3px solid #4CAF50;
      padding-left: 10px;
      background-color: white;
    }
    .log-entry.error {
      border-left-color: #f44336;
      background-color: #ffebee;
    }
    .token-display {
      background-color: #f8f9fa;
      padding: 10px;
      border-radius: 5px;
      font-family: monospace;
      font-size: 11px;
      word-break: break-all;
      margin-top: 10px;
    }
  </style>
</head>
<body>
  <div class="container">
    <h1>🔐 Socket.IO - ミドルウェア（認証とログ）</h1>

    <div class="grid">
      <!-- 接続設定 -->
      <div class="card">
        <h2>接続設定</h2>

        <div style="margin-bottom: 20px;">
          <strong>接続タイプ:</strong>
          <div style="margin-top: 10px;">
            <label>
              <input type="radio" name="connType" value="guest" checked>
              ゲスト（トークンなし）
            </label>
          </div>
          <div>
            <label>
              <input type="radio" name="connType" value="user">
              一般ユーザー（トークンあり）
            </label>
          </div>
          <div>
            <label>
              <input type="radio" name="connType" value="admin">
              管理者（トークンあり）
            </label>
          </div>
        </div>

        <div class="input-group">
          <input type="text" id="username" placeholder="ユーザー名" value="太郎">
        </div>

        <div class="input-group">
          <button onclick="getToken()">トークン取得</button>
          <button onclick="connect()" class="secondary">接続</button>
          <button onclick="disconnect()" class="danger">切断</button>
        </div>

        <div id="tokenDisplay" class="token-display" style="display: none;"></div>
      </div>

      <!-- 接続状態 -->
      <div class="card">
        <h2>接続状態</h2>

        <div id="userInfo" class="user-info guest">
          <div><strong>状態:</strong> <span id="status" class="status disconnected">未接続</span></div>
          <div style="margin-top: 10px;"><strong>ユーザー名:</strong> <span id="displayUsername">-</span></div>
          <div><strong>ロール:</strong> <span id="displayRole">-</span></div>
          <div><strong>Socket ID:</strong> <span id="displaySocketId" style="font-size: 11px; word-break: break-all;">-</span></div>
          <div><strong>接続時刻:</strong> <span id="displayConnectedAt">-</span></div>
        </div>

        <div style="margin-top: 15px;">
          <button onclick="testRateLimit()" class="secondary">レート制限をテスト</button>
        </div>
      </div>

      <!-- ログ -->
      <div class="card">
        <h2>接続ログ</h2>
        <div id="log" class="log"></div>
        <div style="margin-top: 10px;">
          <button onclick="clearLog()" class="danger">ログクリア</button>
          <button onclick="getServerLogs()" class="secondary">サーバーログ取得</button>
        </div>
      </div>

      <!-- 管理者機能 -->
      <div class="card">
        <h2>管理者機能</h2>
        <div id="adminPanel">
          <p style="color: #999; font-style: italic;">管理者として接続すると使用できます</p>
        </div>
      </div>
    </div>
  </div>

  <script src="/socket.io/socket.io.js"></script>
  <script>
    let socket = null;
    let currentToken = null;
    let currentRole = 'guest';

    function addLog(message, isError = false) {
      const log = document.getElementById('log');
      const entry = document.createElement('div');
      entry.className = isError ? 'log-entry error' : 'log-entry';
      const time = new Date().toLocaleTimeString();
      entry.textContent = `[${time}] ${message}`;
      log.appendChild(entry);
      log.scrollTop = log.scrollHeight;
    }

    function clearLog() {
      document.getElementById('log').innerHTML = '';
    }

    function updateUI(connected, userData = null) {
      const status = document.getElementById('status');
      const userInfo = document.getElementById('userInfo');

      if (connected && userData) {
        status.textContent = '接続中';
        status.className = 'status connected';

        document.getElementById('displayUsername').textContent = userData.username;
        document.getElementById('displayRole').textContent = userData.role;
        document.getElementById('displaySocketId').textContent = socket.id;
        document.getElementById('displayConnectedAt').textContent =
          new Date(userData.connectedAt).toLocaleString();

        userInfo.className = `user-info ${userData.role}`;

        // 管理者の場合、管理機能を表示
        if (userData.role === 'admin') {
          document.getElementById('adminPanel').innerHTML = `
            <button onclick="getStats()" class="secondary">統計取得</button>
            <button onclick="getConnectionLogs()">接続ログ取得</button>
            <div id="statsDisplay" style="margin-top: 15px;"></div>
          `;
        }
      } else {
        status.textContent = '未接続';
        status.className = 'status disconnected';
        document.getElementById('displayUsername').textContent = '-';
        document.getElementById('displayRole').textContent = '-';
        document.getElementById('displaySocketId').textContent = '-';
        document.getElementById('displayConnectedAt').textContent = '-';
        userInfo.className = 'user-info guest';

        document.getElementById('adminPanel').innerHTML =
          '<p style="color: #999; font-style: italic;">管理者として接続すると使用できます</p>';
      }
    }

    async function getToken() {
      const type = document.querySelector('input[name="connType"]:checked').value;
      const username = document.getElementById('username').value || '太郎';

      if (type === 'guest') {
        currentToken = null;
        currentRole = 'guest';
        document.getElementById('tokenDisplay').style.display = 'none';
        addLog('ゲストモード（トークンなし）');
        return;
      }

      try {
        const response = await fetch(`/api/token?role=${type}&username=${username}`);
        const data = await response.json();

        currentToken = data.token;
        currentRole = data.role;

        document.getElementById('tokenDisplay').textContent =
          `トークン: ${data.token}\nロール: ${data.role}\nユーザー名: ${data.username}`;
        document.getElementById('tokenDisplay').style.display = 'block';

        addLog(`✅ トークン取得成功（ロール: ${data.role}）`);
      } catch (err) {
        addLog(`❌ トークン取得失敗: ${err.message}`, true);
      }
    }

    function connect() {
      if (socket && socket.connected) {
        addLog('⚠️ 既に接続されています');
        return;
      }

      addLog('📡 接続中...');

      const options = {};

      if (currentToken) {
        options.auth = { token: currentToken };
        addLog(`🔑 認証トークンを使用`);
      }

      socket = io('/', options);

      socket.on('connect', () => {
        addLog('✅ 接続成功');
      });

      socket.on('authenticated', (data) => {
        addLog(`👤 認証完了: ${data.user.username} (${data.user.role})`);
        updateUI(true, {
          username: data.user.username,
          role: data.user.role,
          connectedAt: data.connectedAt
        });
      });

      socket.on('connect_error', (err) => {
        addLog(`❌ 接続エラー: ${err.message}`, true);
        updateUI(false);
      });

      socket.on('error', (data) => {
        addLog(`❌ エラー: ${data.message}`, true);
      });

      socket.on('disconnect', (reason) => {
        addLog(`🔌 切断: ${reason}`);
        updateUI(false);
      });

      socket.on('connectionLogs', (logs) => {
        addLog(`📋 接続ログ受信: ${logs.length}件`);
        console.log('接続ログ:', logs);
      });
    }

    function disconnect() {
      if (socket) {
        socket.disconnect();
        socket = null;
        addLog('🔌 切断しました');
        updateUI(false);
      }
    }

    async function testRateLimit() {
      addLog('⏱️ レート制限テスト開始（5回連続接続）');

      for (let i = 1; i <= 5; i++) {
        addLog(`接続試行 ${i}/5...`);

        const testSocket = io('/', {
          auth: currentToken ? { token: currentToken } : {}
        });

        await new Promise(resolve => {
          testSocket.on('connect', () => {
            addLog(`  ✅ 接続 ${i} 成功`);
            testSocket.disconnect();
            resolve();
          });

          testSocket.on('connect_error', (err) => {
            addLog(`  ❌ 接続 ${i} 失敗: ${err.message}`, true);
            resolve();
          });
        });

        await new Promise(resolve => setTimeout(resolve, 500));
      }

      addLog('テスト完了');
    }

    async function getServerLogs() {
      try {
        const response = await fetch('/api/logs');
        const logs = await response.json();
        addLog(`📋 サーバーログ取得: ${logs.length}件`);
        console.log('サーバーログ:', logs);
      } catch (err) {
        addLog(`❌ ログ取得失敗: ${err.message}`, true);
      }
    }

    function getConnectionLogs() {
      if (!socket || !socket.connected) {
        addLog('⚠️ 接続してください');
        return;
      }

      socket.emit('getConnectionLogs');
    }

    function getStats() {
      if (!socket || !socket.connected) {
        addLog('⚠️ 接続してください');
        return;
      }

      // 管理者ネームスペースに接続
      const adminSocket = io('/admin', {
        auth: { token: currentToken }
      });

      adminSocket.on('connect', () => {
        addLog('👑 管理者ネームスペースに接続');
        adminSocket.emit('getStats');
      });

      adminSocket.on('connect_error', (err) => {
        addLog(`❌ 管理者接続失敗: ${err.message}`, true);
      });

      adminSocket.on('stats', (data) => {
        addLog(`📊 統計取得: 合計=${data.totalConnections}, 管理者=${data.adminConnections}, ログ=${data.logs}`);

        document.getElementById('statsDisplay').innerHTML = `
          <div style="background-color: #f8f9fa; padding: 10px; border-radius: 5px; margin-top: 10px;">
            <div>合計接続数: <strong>${data.totalConnections}</strong></div>
            <div>管理者接続数: <strong>${data.adminConnections}</strong></div>
            <div>ログ件数: <strong>${data.logs}</strong></div>
          </div>
        `;

        adminSocket.disconnect();
      });
    }
  </script>
</body>
</html>
```

---

## 🔍 ミドルウェアAPIの詳細

### 基本形

```javascript
io.use((socket, next) => {
  // 処理を実行
  if (条件) {
    next();  // 次のミドルウェアへ
  } else {
    next(new Error('エラーメッセージ'));  // 接続拒否
  }
});
```

### 非同期ミドルウェア

```javascript
io.use(async (socket, next) => {
  try {
    const result = await someAsyncOperation();
    socket.data.result = result;
    next();
  } catch (error) {
    next(new Error('非同期処理失敗'));
  }
});
```

### データの保存

```javascript
io.use((socket, next) => {
  // socket.data に保存（接続中ずっと利用可能）
  socket.data.customData = { ... };
  next();
});

io.on('connection', (socket) => {
  console.log(socket.data.customData);  // アクセス可能
});
```

---

## 🧪 動作確認

### テスト手順

1. サーバー起動: `node server.js`
2. ブラウザで `http://localhost:3000` を開く
3. ゲストとして接続 → 成功することを確認
4. トークン取得（一般ユーザー） → 接続成功を確認
5. トークン取得（管理者） → 管理機能が表示されることを確認
6. レート制限テスト → 5秒以内の再接続が拒否されることを確認

---

## 📝 練習課題

### 初級
1. **IPホワイトリスト**: 特定のIPのみ接続を許可
2. **User-Agent検証**: 特定のブラウザのみ許可

### 中級
3. **データベース認証**: ユーザー情報をDBから取得して認証
4. **セッション管理**: Cookie/Sessionベースの認証

### 上級
5. **OAuth認証**: Google/GitHub等のOAuth連携
6. **動的レート制限**: ユーザーの役割に応じて制限を変更

---

## 💡 次のステップ

ミドルウェアを理解したら、次は「エラーハンドリング」を学んで、堅牢なアプリケーションを作りましょう！

👉 [Lesson 8: エラーハンドリング](./08-error-handling.md)

---

## 📚 参考リソース

- [Socket.IO - Middlewares](https://socket.io/docs/v4/middlewares/)
- [Socket.IO - Authentication](https://socket.io/docs/v4/middlewares/#sending-credentials)
