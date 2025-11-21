# Lesson 8: エラーハンドリング

## 🎯 学習目標

- Socket.IOにおけるエラーの種類を理解する
- 適切なエラーハンドリングを実装する
- エラーをクライアントに通知する方法を学ぶ
- エラーログを記録する方法を学ぶ

---

## 📋 メリット

✅ **堅牢性**: アプリケーションのクラッシュを防ぐ
✅ **デバッグ**: エラーの原因を特定しやすい
✅ **ユーザー体験**: 適切なエラーメッセージを表示できる
✅ **監視**: エラーを記録して分析できる
✅ **回復**: エラーから自動的に復旧できる

---

## ⚠️ デメリット

❌ **複雑性**: エラーハンドリングコードが増える
❌ **パフォーマンス**: try-catchのオーバーヘッド
❌ **情報漏洩**: エラーメッセージで内部情報が漏れる可能性
❌ **過剰処理**: すべてのエラーをキャッチすると本来の問題を隠す

---

## ⚙️ 技術的原理

### エラーの種類

```
1. 接続エラー (connect_error)
   - 認証失敗
   - ミドルウェアでの拒否
   - ネットワークエラー

2. トランスポートエラー (transport_error)
   - WebSocketの切断
   - HTTPポーリングの失敗

3. イベントハンドラーエラー
   - イベントリスナー内の例外
   - 非同期処理のエラー

4. タイムアウトエラー
   - ping/pongのタイムアウト
   - ACKのタイムアウト

5. アプリケーションエラー
   - ビジネスロジックのエラー
   - データ検証エラー
```

### エラー伝播のフロー

```
エラー発生
    ↓
try-catchでキャッチ
    ↓
エラーログを記録
    ↓
クライアントにエラー通知
    ↓
適切な復旧処理
```

---

## 💼 ユースケース

### 適している場面

1. **本番環境**: すべてのエラーを適切に処理
2. **ユーザー入力**: 検証エラーを分かりやすく通知
3. **外部API**: タイムアウトやレート制限のエラー
4. **データベース**: クエリエラーの処理
5. **監視システム**: エラー率を追跡

### 注意が必要な場面

1. **開発環境**: 詳細なスタックトレースが必要
2. **デバッグ**: エラーを隠さず表示
3. **パフォーマンステスト**: エラーハンドリングのオーバーヘッド測定

---

## 💻 実装例

### ステップ1: プロジェクトのセットアップ

```bash
mkdir socket-error-handling
cd socket-error-handling
npm init -y
npm install express socket.io winston
```

### ステップ2: サーバー側の実装

`server.js`:

```javascript
const express = require('express');
const { createServer } = require('http');
const { Server } = require('socket.io');
const winston = require('winston');

const app = express();
const httpServer = createServer(app);
const io = new Server(httpServer);

app.use(express.static('public'));

// ===== ロガーの設定 =====

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.colorize(),
        winston.format.simple()
      )
    }),
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' })
  ]
});

// ===== エラー統計 =====

const errorStats = {
  connection: 0,
  transport: 0,
  handler: 0,
  validation: 0,
  timeout: 0,
  total: 0
};

function recordError(type, error, socket) {
  errorStats[type]++;
  errorStats.total++;

  logger.error('Socket.IO Error', {
    type: type,
    message: error.message,
    stack: error.stack,
    socketId: socket?.id,
    timestamp: new Date().toISOString()
  });
}

// ===== グローバルエラーハンドラー =====

process.on('uncaughtException', (error) => {
  logger.error('Uncaught Exception', {
    message: error.message,
    stack: error.stack
  });
  // 本番環境では適切に終了処理を行う
  // process.exit(1);
});

process.on('unhandledRejection', (reason, promise) => {
  logger.error('Unhandled Rejection', {
    reason: reason,
    promise: promise
  });
});

// ===== ミドルウェアでのエラーハンドリング =====

io.use((socket, next) => {
  try {
    const token = socket.handshake.auth.token;

    // 認証チェック（デモ用）
    if (token && token !== 'valid-token') {
      const error = new Error('無効なトークンです');
      error.data = { code: 'AUTH_FAILED', providedToken: token };
      recordError('connection', error, socket);
      return next(error);
    }

    next();
  } catch (error) {
    recordError('connection', error, socket);
    next(error);
  }
});

// ===== 接続処理 =====

io.on('connection', (socket) => {
  logger.info('Client connected', { socketId: socket.id });

  // ===== 接続エラー =====

  socket.on('connect_error', (error) => {
    recordError('connection', error, socket);
    socket.emit('error', {
      type: 'connection',
      message: '接続エラーが発生しました'
    });
  });

  // ===== イベントハンドラーのエラー =====

  socket.on('processData', (data, callback) => {
    try {
      // データ検証
      if (!data || typeof data !== 'object') {
        throw new Error('無効なデータ形式です');
      }

      if (!data.value || data.value < 0) {
        const error = new Error('値は0以上である必要があります');
        error.code = 'VALIDATION_ERROR';
        throw error;
      }

      // 処理実行（デモ用に意図的にエラーを発生させる）
      if (data.value === 999) {
        throw new Error('値999は処理できません');
      }

      // 成功
      const result = {
        success: true,
        processed: data.value * 2,
        timestamp: new Date().toISOString()
      };

      if (callback) callback(null, result);
      socket.emit('processComplete', result);

    } catch (error) {
      recordError('validation', error, socket);

      // クライアントにエラーを通知
      const errorResponse = {
        success: false,
        error: {
          message: error.message,
          code: error.code || 'PROCESSING_ERROR',
          timestamp: new Date().toISOString()
        }
      };

      if (callback) callback(errorResponse);
      socket.emit('processError', errorResponse);
    }
  });

  // ===== 非同期処理のエラー =====

  socket.on('fetchData', async (params, callback) => {
    try {
      // 非同期処理のシミュレーション
      const data = await simulateAsyncOperation(params);

      callback({ success: true, data: data });

    } catch (error) {
      recordError('handler', error, socket);

      callback({
        success: false,
        error: {
          message: error.message,
          code: 'ASYNC_ERROR'
        }
      });
    }
  });

  // ===== タイムアウトエラー =====

  socket.on('longOperation', (params, callback) => {
    const timeoutMs = params.timeout || 5000;
    let completed = false;

    // タイムアウト設定
    const timeoutId = setTimeout(() => {
      if (!completed) {
        completed = true;
        const error = new Error('操作がタイムアウトしました');
        recordError('timeout', error, socket);

        callback({
          success: false,
          error: {
            message: error.message,
            code: 'TIMEOUT'
          }
        });
      }
    }, timeoutMs);

    // 実際の処理（シミュレーション）
    setTimeout(() => {
      if (!completed) {
        completed = true;
        clearTimeout(timeoutId);

        callback({
          success: true,
          message: '操作が完了しました'
        });
      }
    }, params.delay || 3000);
  });

  // ===== エラー再現（デモ用） =====

  socket.on('triggerError', (errorType) => {
    logger.info('Triggering error', { type: errorType, socketId: socket.id });

    try {
      switch (errorType) {
        case 'throw':
          throw new Error('意図的なエラー（throw）');

        case 'reference':
          // 存在しない変数を参照
          console.log(nonExistentVariable);
          break;

        case 'type':
          // 型エラー
          null.toString();
          break;

        case 'async':
          // 非同期エラー
          Promise.reject(new Error('非同期エラー'));
          break;

        default:
          socket.emit('error', {
            message: '不明なエラータイプです'
          });
      }
    } catch (error) {
      recordError('handler', error, socket);

      socket.emit('error', {
        type: 'triggered',
        message: error.message,
        stack: process.env.NODE_ENV === 'development' ? error.stack : undefined
      });
    }
  });

  // ===== エラー統計の取得 =====

  socket.on('getErrorStats', () => {
    socket.emit('errorStats', errorStats);
  });

  // ===== トランスポートエラー =====

  socket.conn.on('error', (error) => {
    recordError('transport', error, socket);
    logger.error('Transport error', {
      socketId: socket.id,
      error: error.message
    });
  });

  // ===== 切断エラー =====

  socket.on('disconnect', (reason) => {
    logger.info('Client disconnected', {
      socketId: socket.id,
      reason: reason
    });

    if (reason === 'transport error' || reason === 'transport close') {
      recordError('transport', new Error(`Disconnect: ${reason}`), socket);
    }
  });

  socket.on('error', (error) => {
    recordError('handler', error, socket);
    logger.error('Socket error', {
      socketId: socket.id,
      error: error.message
    });
  });
});

// ===== 非同期処理のシミュレーション =====

function simulateAsyncOperation(params) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      if (params.fail) {
        reject(new Error('非同期処理が失敗しました'));
      } else {
        resolve({
          id: Math.random().toString(36).substr(2, 9),
          value: params.value || 'default',
          timestamp: new Date().toISOString()
        });
      }
    }, 1000);
  });
}

// ===== API エンドポイント =====

app.get('/api/errors', (req, res) => {
  res.json(errorStats);
});

app.get('/api/logs', (req, res) => {
  // 実際のアプリではログファイルを読み込む
  res.json({ message: 'See error.log and combined.log files' });
});

const PORT = 3000;
httpServer.listen(PORT, () => {
  logger.info(`Server started on port ${PORT}`);
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
  <title>Socket.IO - エラーハンドリング</title>
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
    .button-group {
      display: grid;
      grid-template-columns: repeat(2, 1fr);
      gap: 10px;
      margin-bottom: 15px;
    }
    button {
      padding: 12px;
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
    input[type="number"] {
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
      height: 300px;
      overflow-y: auto;
      font-family: monospace;
      font-size: 12px;
    }
    .log-entry {
      margin: 5px 0;
      padding: 8px;
      border-radius: 3px;
      border-left: 3px solid #4CAF50;
      background-color: white;
    }
    .log-entry.error {
      border-left-color: #f44336;
      background-color: #ffebee;
      color: #c62828;
    }
    .log-entry.warning {
      border-left-color: #ff9800;
      background-color: #fff3e0;
      color: #e65100;
    }
    .log-entry.success {
      border-left-color: #4CAF50;
      background-color: #e8f5e9;
      color: #2e7d32;
    }
    .stats {
      display: grid;
      grid-template-columns: repeat(3, 1fr);
      gap: 10px;
      margin-top: 15px;
    }
    .stat-card {
      background-color: #f8f9fa;
      padding: 15px;
      border-radius: 5px;
      text-align: center;
    }
    .stat-number {
      font-size: 32px;
      font-weight: bold;
      color: #f44336;
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
    <h1>⚠️ Socket.IO - エラーハンドリング</h1>

    <div class="grid">
      <!-- 接続エラー -->
      <div class="card">
        <h2>🔌 接続エラー</h2>
        <div class="button-group">
          <button onclick="connectValid()">正常接続</button>
          <button onclick="connectInvalid()" class="danger">無効トークンで接続</button>
        </div>
        <div style="margin-top: 10px;">
          <button onclick="disconnect()" class="warning">切断</button>
        </div>
      </div>

      <!-- データ検証エラー -->
      <div class="card">
        <h2>✅ データ検証エラー</h2>
        <input type="number" id="valueInput" placeholder="値を入力（0以上）" value="10">
        <div class="button-group">
          <button onclick="processData()">正常データ送信</button>
          <button onclick="processInvalidData()" class="danger">無効データ送信</button>
          <button onclick="processSpecialValue()" class="warning">特殊値（999）送信</button>
          <button onclick="processNegative()" class="danger">負の値送信</button>
        </div>
      </div>

      <!-- 非同期エラー -->
      <div class="card">
        <h2>⏱️ 非同期・タイムアウトエラー</h2>
        <div class="button-group">
          <button onclick="fetchDataSuccess()">非同期処理（成功）</button>
          <button onclick="fetchDataFail()" class="danger">非同期処理（失敗）</button>
          <button onclick="longOperationSuccess()">長時間処理（成功）</button>
          <button onclick="longOperationTimeout()" class="warning">長時間処理（タイムアウト）</button>
        </div>
      </div>

      <!-- エラー再現 -->
      <div class="card">
        <h2>💥 エラー再現（デモ用）</h2>
        <div class="button-group">
          <button onclick="triggerThrowError()" class="danger">Throw Error</button>
          <button onclick="triggerReferenceError()" class="danger">Reference Error</button>
          <button onclick="triggerTypeError()" class="danger">Type Error</button>
          <button onclick="triggerAsyncError()" class="danger">Async Error</button>
        </div>
      </div>

      <!-- エラー統計 -->
      <div class="card">
        <h2>📊 エラー統計</h2>
        <button onclick="getErrorStats()" style="width: 100%; margin-bottom: 15px;">
          統計を更新
        </button>
        <div id="statsContainer" class="stats">
          <div class="stat-card">
            <div class="stat-number" id="stat-connection">0</div>
            <div class="stat-label">接続エラー</div>
          </div>
          <div class="stat-card">
            <div class="stat-number" id="stat-validation">0</div>
            <div class="stat-label">検証エラー</div>
          </div>
          <div class="stat-card">
            <div class="stat-number" id="stat-handler">0</div>
            <div class="stat-label">ハンドラーエラー</div>
          </div>
          <div class="stat-card">
            <div class="stat-number" id="stat-timeout">0</div>
            <div class="stat-label">タイムアウト</div>
          </div>
          <div class="stat-card">
            <div class="stat-number" id="stat-transport">0</div>
            <div class="stat-label">トランスポート</div>
          </div>
          <div class="stat-card">
            <div class="stat-number" id="stat-total">0</div>
            <div class="stat-label">合計</div>
          </div>
        </div>
      </div>

      <!-- ログ -->
      <div class="card" style="grid-column: 1 / -1;">
        <h2>📋 イベントログ</h2>
        <div id="log" class="log"></div>
        <button onclick="clearLog()" class="danger" style="margin-top: 10px; width: 100%;">
          ログクリア
        </button>
      </div>
    </div>
  </div>

  <script src="/socket.io/socket.io.js"></script>
  <script>
    let socket = null;

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

    function setupSocketListeners(sock) {
      sock.on('connect', () => {
        addLog('✅ 接続成功', 'success');
      });

      sock.on('connect_error', (error) => {
        addLog(`❌ 接続エラー: ${error.message}`, 'error');
      });

      sock.on('error', (data) => {
        addLog(`❌ エラー: ${data.message}`, 'error');
        if (data.stack) {
          console.error(data.stack);
        }
      });

      sock.on('processComplete', (data) => {
        addLog(`✅ 処理完了: ${JSON.stringify(data)}`, 'success');
      });

      sock.on('processError', (data) => {
        addLog(`❌ 処理エラー: ${data.error.message} (${data.error.code})`, 'error');
      });

      sock.on('errorStats', (stats) => {
        addLog(`📊 統計更新: 合計${stats.total}件のエラー`, 'info');
        updateStats(stats);
      });

      sock.on('disconnect', (reason) => {
        addLog(`🔌 切断: ${reason}`, 'warning');
      });
    }

    function updateStats(stats) {
      document.getElementById('stat-connection').textContent = stats.connection;
      document.getElementById('stat-validation').textContent = stats.validation;
      document.getElementById('stat-handler').textContent = stats.handler;
      document.getElementById('stat-timeout').textContent = stats.timeout;
      document.getElementById('stat-transport').textContent = stats.transport;
      document.getElementById('stat-total').textContent = stats.total;
    }

    function connectValid() {
      if (socket) socket.disconnect();

      addLog('🔌 正常接続を試行...', 'info');
      socket = io('/', {
        auth: { token: 'valid-token' }
      });
      setupSocketListeners(socket);
    }

    function connectInvalid() {
      if (socket) socket.disconnect();

      addLog('🔌 無効トークンで接続を試行...', 'warning');
      socket = io('/', {
        auth: { token: 'invalid-token' }
      });
      setupSocketListeners(socket);
    }

    function disconnect() {
      if (socket) {
        socket.disconnect();
        socket = null;
        addLog('🔌 切断しました', 'info');
      }
    }

    function processData() {
      if (!socket || !socket.connected) {
        addLog('⚠️ 先に接続してください', 'warning');
        return;
      }

      const value = parseInt(document.getElementById('valueInput').value) || 10;
      addLog(`📤 データ送信: value=${value}`, 'info');

      socket.emit('processData', { value: value }, (error, result) => {
        if (error) {
          addLog(`❌ ACKエラー: ${error.error.message}`, 'error');
        } else {
          addLog(`✅ ACK成功: ${JSON.stringify(result)}`, 'success');
        }
      });
    }

    function processInvalidData() {
      if (!socket || !socket.connected) {
        addLog('⚠️ 先に接続してください', 'warning');
        return;
      }

      addLog('📤 無効データ送信: null', 'warning');
      socket.emit('processData', null);
    }

    function processSpecialValue() {
      if (!socket || !socket.connected) {
        addLog('⚠️ 先に接続してください', 'warning');
        return;
      }

      addLog('📤 特殊値送信: value=999', 'warning');
      socket.emit('processData', { value: 999 });
    }

    function processNegative() {
      if (!socket || !socket.connected) {
        addLog('⚠️ 先に接続してください', 'warning');
        return;
      }

      addLog('📤 負の値送信: value=-10', 'warning');
      socket.emit('processData', { value: -10 });
    }

    function fetchDataSuccess() {
      if (!socket || !socket.connected) {
        addLog('⚠️ 先に接続してください', 'warning');
        return;
      }

      addLog('📤 非同期処理開始（成功ケース）', 'info');
      socket.emit('fetchData', { value: 'test', fail: false }, (response) => {
        if (response.success) {
          addLog(`✅ 非同期処理成功: ${JSON.stringify(response.data)}`, 'success');
        } else {
          addLog(`❌ 非同期処理失敗: ${response.error.message}`, 'error');
        }
      });
    }

    function fetchDataFail() {
      if (!socket || !socket.connected) {
        addLog('⚠️ 先に接続してください', 'warning');
        return;
      }

      addLog('📤 非同期処理開始（失敗ケース）', 'warning');
      socket.emit('fetchData', { value: 'test', fail: true }, (response) => {
        if (response.success) {
          addLog(`✅ 非同期処理成功: ${JSON.stringify(response.data)}`, 'success');
        } else {
          addLog(`❌ 非同期処理失敗: ${response.error.message}`, 'error');
        }
      });
    }

    function longOperationSuccess() {
      if (!socket || !socket.connected) {
        addLog('⚠️ 先に接続してください', 'warning');
        return;
      }

      addLog('📤 長時間処理開始（3秒、タイムアウト5秒）', 'info');
      socket.emit('longOperation', { delay: 3000, timeout: 5000 }, (response) => {
        if (response.success) {
          addLog(`✅ 処理完了: ${response.message}`, 'success');
        } else {
          addLog(`❌ 処理失敗: ${response.error.message}`, 'error');
        }
      });
    }

    function longOperationTimeout() {
      if (!socket || !socket.connected) {
        addLog('⚠️ 先に接続してください', 'warning');
        return;
      }

      addLog('📤 長時間処理開始（6秒、タイムアウト3秒）', 'warning');
      socket.emit('longOperation', { delay: 6000, timeout: 3000 }, (response) => {
        if (response.success) {
          addLog(`✅ 処理完了: ${response.message}`, 'success');
        } else {
          addLog(`❌ タイムアウト: ${response.error.message}`, 'error');
        }
      });
    }

    function triggerThrowError() {
      if (!socket || !socket.connected) {
        addLog('⚠️ 先に接続してください', 'warning');
        return;
      }

      addLog('💥 Throw Errorを発生させます', 'warning');
      socket.emit('triggerError', 'throw');
    }

    function triggerReferenceError() {
      if (!socket || !socket.connected) {
        addLog('⚠️ 先に接続してください', 'warning');
        return;
      }

      addLog('💥 Reference Errorを発生させます', 'warning');
      socket.emit('triggerError', 'reference');
    }

    function triggerTypeError() {
      if (!socket || !socket.connected) {
        addLog('⚠️ 先に接続してください', 'warning');
        return;
      }

      addLog('💥 Type Errorを発生させます', 'warning');
      socket.emit('triggerError', 'type');
    }

    function triggerAsyncError() {
      if (!socket || !socket.connected) {
        addLog('⚠️ 先に接続してください', 'warning');
        return;
      }

      addLog('💥 Async Errorを発生させます', 'warning');
      socket.emit('triggerError', 'async');
    }

    function getErrorStats() {
      if (!socket || !socket.connected) {
        addLog('⚠️ 先に接続してください', 'warning');
        return;
      }

      socket.emit('getErrorStats');
    }

    // 自動接続
    window.addEventListener('load', () => {
      connectValid();
    });
  </script>
</body>
</html>
```

---

## 🔍 エラーハンドリングのベストプラクティス

### 1. 適切なエラーメッセージ

```javascript
// ❌ 悪い例
throw new Error('エラー');

// ✅ 良い例
throw new Error('ユーザーID 123 が見つかりません');
```

### 2. エラーコードの使用

```javascript
const error = new Error('認証に失敗しました');
error.code = 'AUTH_FAILED';
error.statusCode = 401;
throw error;
```

### 3. ログの記録

```javascript
socket.on('error', (error) => {
  logger.error('Socket error', {
    socketId: socket.id,
    error: error.message,
    stack: error.stack,
    user: socket.data.user
  });
});
```

### 4. クライアントへの通知

```javascript
// セキュリティを考慮したエラーメッセージ
const userFriendlyError = {
  message: '処理中にエラーが発生しました',
  code: error.code,
  // スタックトレースは本番環境では送信しない
  stack: process.env.NODE_ENV === 'development' ? error.stack : undefined
};

socket.emit('error', userFriendlyError);
```

---

## 📝 練習課題

### 初級
1. **カスタムエラークラス**: 独自のエラークラスを作成
2. **エラー通知**: エラー発生時にメール通知

### 中級
3. **リトライ機構**: 一時的なエラーを自動リトライ
4. **エラー集約**: 同じエラーをまとめて記録

### 上級
5. **エラー監視**: Sentry等の監視サービスと統合
6. **サーキットブレーカー**: 連続エラー時にサービスを停止

---

## 💡 次のステップ

エラーハンドリングをマスターしたら、次は「再接続とバッファリング」を学んで、接続の信頼性を向上させましょう！

👉 [Lesson 9: 再接続とバッファリング](./09-reconnection.md)

---

## 📚 参考リソース

- [Socket.IO - Error handling](https://socket.io/docs/v4/error-handling/)
- [Node.js - Error handling best practices](https://nodejs.org/en/docs/guides/error-handling/)
