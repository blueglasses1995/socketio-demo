# Lesson 2: 基本的な接続

## 🎯 学習目標

- Socket.IOサーバーをセットアップする
- クライアントからサーバーに接続する
- 接続イベントを処理する
- 切断イベントを処理する

---

## 📋 メリット

✅ **シンプルなAPI**: わずか数行のコードで接続を確立できる
✅ **自動管理**: 接続の確立、維持、切断を自動で処理
✅ **セッション管理**: 各クライアントに一意のIDが自動割り当て
✅ **イベントベース**: 直感的なイベント駆動型プログラミング

---

## ⚠️ デメリット

❌ **接続オーバーヘッド**: HTTP接続確立後、WebSocketへのアップグレードが必要
❌ **リソース消費**: 各接続がサーバーのメモリを消費
❌ **接続数制限**: サーバーのリソースに応じて同時接続数に上限がある
❌ **ファイアウォール問題**: 企業ネットワークでブロックされる可能性

---

## ⚙️ 技術的原理

### 接続確立のフロー

```
1. クライアントがサーバーにHTTP接続要求
   ↓
2. サーバーがセッションIDを生成・返却
   ↓
3. クライアントがWebSocketアップグレードを要求
   ↓
4. サーバーがアップグレードを承認
   ↓
5. WebSocket接続確立（双方向通信開始）
   ↓
6. 'connection'イベントが発火
```

### 内部で何が起きているか

```javascript
// クライアント側の内部処理
1. new Manager() が作成される
2. Engine.IO トランスポートが初期化される
3. HTTP POSTで handshake リクエスト送信
4. サーバーから sid (session ID) を受信
5. WebSocket接続試行
6. 成功したら 'connect' イベント発火

// サーバー側の内部処理
1. HTTP サーバーが起動
2. Socket.IO が HTTP サーバーにアタッチ
3. クライアントからの接続要求を待機
4. 接続要求を受信したら Engine.IO で処理
5. Socket インスタンスを作成
6. 'connection' イベント発火
```

---

## 💼 ユースケース

### 適している場面

1. **リアルタイム通知システム**: ユーザーがログインしたら接続を維持
2. **チャットアプリ**: ユーザーがチャット画面を開いたら接続
3. **ライブダッシュボード**: データ監視画面で常時接続
4. **オンラインゲーム**: ゲーム開始時に接続確立

### 注意が必要な場面

1. **大規模システム**: 数万〜数十万の同時接続（スケーリング必要）
2. **短命な接続**: 一度きりのデータ取得（HTTP APIの方が適切）
3. **大量データ転送**: ファイルアップロード等（HTTP APIの方が適切）

---

## 💻 実装例

### ステップ1: プロジェクトのセットアップ

```bash
# プロジェクトディレクトリを作成
mkdir socket-basic-connection
cd socket-basic-connection

# package.jsonを初期化
npm init -y

# 必要なパッケージをインストール
npm install express socket.io
```

### ステップ2: サーバー側の実装

`server.js` を作成:

```javascript
const express = require('express');
const { createServer } = require('http');
const { Server } = require('socket.io');

// Expressアプリケーションを作成
const app = express();
// HTTPサーバーを作成
const httpServer = createServer(app);
// Socket.IOサーバーを作成（HTTPサーバーにアタッチ）
const io = new Server(httpServer);

// 静的ファイルを配信（HTMLファイル等）
app.use(express.static('public'));

// クライアントが接続したときの処理
io.on('connection', (socket) => {
  console.log('🔌 クライアントが接続しました');
  console.log(`   Socket ID: ${socket.id}`);
  console.log(`   接続時刻: ${new Date().toLocaleTimeString()}`);

  // クライアント側に接続成功メッセージを送信
  socket.emit('welcome', {
    message: 'サーバーへの接続に成功しました！',
    socketId: socket.id,
    timestamp: new Date().toISOString()
  });

  // クライアントが切断したときの処理
  socket.on('disconnect', (reason) => {
    console.log('🔌 クライアントが切断されました');
    console.log(`   Socket ID: ${socket.id}`);
    console.log(`   理由: ${reason}`);
    console.log(`   切断時刻: ${new Date().toLocaleTimeString()}`);
  });
});

// サーバーを起動
const PORT = 3000;
httpServer.listen(PORT, () => {
  console.log(`🚀 サーバーが起動しました: http://localhost:${PORT}`);
});
```

### ステップ3: クライアント側の実装

`public/index.html` を作成:

```html
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Socket.IO - 基本的な接続</title>
  <style>
    body {
      font-family: Arial, sans-serif;
      max-width: 800px;
      margin: 50px auto;
      padding: 20px;
      background-color: #f5f5f5;
    }
    .container {
      background-color: white;
      border-radius: 8px;
      padding: 30px;
      box-shadow: 0 2px 10px rgba(0,0,0,0.1);
    }
    h1 {
      color: #333;
      border-bottom: 3px solid #4CAF50;
      padding-bottom: 10px;
    }
    .status {
      padding: 15px;
      border-radius: 5px;
      margin: 20px 0;
      font-weight: bold;
    }
    .connected {
      background-color: #d4edda;
      color: #155724;
      border: 1px solid #c3e6cb;
    }
    .disconnected {
      background-color: #f8d7da;
      color: #721c24;
      border: 1px solid #f5c6cb;
    }
    .info {
      background-color: #f8f9fa;
      padding: 15px;
      border-radius: 5px;
      margin: 10px 0;
    }
    .info-label {
      font-weight: bold;
      color: #666;
    }
    button {
      background-color: #4CAF50;
      color: white;
      padding: 10px 20px;
      border: none;
      border-radius: 5px;
      cursor: pointer;
      font-size: 16px;
      margin: 5px;
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
    #log {
      background-color: #f8f9fa;
      border: 1px solid #ddd;
      border-radius: 5px;
      padding: 15px;
      max-height: 300px;
      overflow-y: auto;
      font-family: monospace;
      font-size: 14px;
    }
    .log-entry {
      margin: 5px 0;
      padding: 5px;
      border-left: 3px solid #4CAF50;
      padding-left: 10px;
    }
  </style>
</head>
<body>
  <div class="container">
    <h1>🔌 Socket.IO - 基本的な接続</h1>

    <div id="status" class="status disconnected">
      ⚫ 未接続
    </div>

    <div class="info">
      <div><span class="info-label">Socket ID:</span> <span id="socketId">-</span></div>
      <div><span class="info-label">接続状態:</span> <span id="connectionState">-</span></div>
    </div>

    <div style="margin: 20px 0;">
      <button onclick="connect()">接続</button>
      <button onclick="disconnect()" class="disconnect">切断</button>
    </div>

    <h3>📋 ログ</h3>
    <div id="log"></div>
  </div>

  <!-- Socket.IOクライアントライブラリを読み込み -->
  <script src="/socket.io/socket.io.js"></script>
  <script>
    let socket = null;

    // ログに追加する関数
    function addLog(message) {
      const log = document.getElementById('log');
      const entry = document.createElement('div');
      entry.className = 'log-entry';
      const timestamp = new Date().toLocaleTimeString();
      entry.textContent = `[${timestamp}] ${message}`;
      log.appendChild(entry);
      log.scrollTop = log.scrollHeight; // 自動スクロール
    }

    // ステータス表示を更新
    function updateStatus(connected) {
      const statusDiv = document.getElementById('status');
      const stateSpan = document.getElementById('connectionState');

      if (connected) {
        statusDiv.className = 'status connected';
        statusDiv.textContent = '🟢 接続中';
        stateSpan.textContent = '接続済み';
      } else {
        statusDiv.className = 'status disconnected';
        statusDiv.textContent = '⚫ 未接続';
        stateSpan.textContent = '切断';
        document.getElementById('socketId').textContent = '-';
      }
    }

    // 接続する関数
    function connect() {
      if (socket && socket.connected) {
        addLog('⚠️ すでに接続されています');
        return;
      }

      addLog('📡 サーバーに接続中...');

      // Socket.IOクライアントを作成して接続
      socket = io();

      // 接続成功時
      socket.on('connect', () => {
        addLog('✅ サーバーに接続しました');
        document.getElementById('socketId').textContent = socket.id;
        updateStatus(true);
      });

      // サーバーからのウェルカムメッセージ
      socket.on('welcome', (data) => {
        addLog(`💬 ${data.message}`);
        addLog(`📌 割り当てられたID: ${data.socketId}`);
      });

      // 切断時
      socket.on('disconnect', (reason) => {
        addLog(`❌ サーバーから切断されました (理由: ${reason})`);
        updateStatus(false);
      });

      // 接続エラー時
      socket.on('connect_error', (error) => {
        addLog(`⚠️ 接続エラー: ${error.message}`);
        updateStatus(false);
      });
    }

    // 切断する関数
    function disconnect() {
      if (!socket || !socket.connected) {
        addLog('⚠️ 接続されていません');
        return;
      }

      addLog('🔌 切断中...');
      socket.disconnect();
    }

    // ページ読み込み時に自動接続
    window.addEventListener('load', () => {
      addLog('🌐 ページが読み込まれました');
      connect();
    });
  </script>
</body>
</html>
```

### ステップ4: ディレクトリ構造

```
socket-basic-connection/
├── server.js          # サーバー側のコード
├── package.json       # プロジェクト設定
└── public/
    └── index.html     # クライアント側のHTML
```

### ステップ5: 実行方法

```bash
# サーバーを起動
node server.js

# ブラウザで開く
# http://localhost:3000
```

---

## 🔍 コードの詳細解説

### サーバー側の重要ポイント

```javascript
// Socket.IOサーバーを作成
const io = new Server(httpServer);
// オプションを指定することも可能
// const io = new Server(httpServer, {
//   cors: {
//     origin: "http://localhost:8080" // CORS設定
//   },
//   pingTimeout: 60000,  // タイムアウト時間
//   pingInterval: 25000  // Ping送信間隔
// });

// 接続イベントのリスナー
io.on('connection', (socket) => {
  // socket: 接続してきたクライアントのSocketインスタンス
  // socket.id: 一意のSocket ID（自動生成）
  // socket.handshake: 接続時のハンドシェイク情報

  console.log(socket.id);  // 例: "Xy3kj2nW_4XnN4XWAAAB"
});
```

### クライアント側の重要ポイント

```javascript
// サーバーに接続
const socket = io();  // 現在のホストに自動接続

// 別のサーバーに接続する場合
// const socket = io('http://localhost:3000');

// オプションを指定
// const socket = io({
//   autoConnect: false,  // 自動接続を無効化
//   reconnection: true,  // 再接続を有効化（デフォルト: true）
//   reconnectionDelay: 1000,  // 再接続の遅延時間
//   reconnectionAttempts: 5   // 再接続試行回数
// });
```

### 切断の理由（disconnect reason）

| 理由 | 説明 |
|-----|------|
| `io server disconnect` | サーバー側から強制切断 |
| `io client disconnect` | クライアント側から切断 |
| `ping timeout` | ハートビートのタイムアウト |
| `transport close` | トランスポート層の切断 |
| `transport error` | トランスポート層のエラー |

---

## 🧪 動作確認

### 1. 正常な接続

1. サーバーを起動: `node server.js`
2. ブラウザで `http://localhost:3000` を開く
3. 自動的に接続される
4. ログに「接続しました」と表示される

### 2. 手動接続・切断

1. 「切断」ボタンをクリック
2. ログに「切断されました」と表示される
3. 「接続」ボタンをクリック
4. 再度接続される

### 3. サーバー側のログ

```
🚀 サーバーが起動しました: http://localhost:3000
🔌 クライアントが接続しました
   Socket ID: Xy3kj2nW_4XnN4XWAAAB
   接続時刻: 14:23:45
```

---

## 🐛 よくある問題とトラブルシューティング

### 問題1: 接続できない

```javascript
// 原因: サーバーが起動していない、またはポートが違う
// 解決策: サーバーのURLとポートを確認

const socket = io('http://localhost:3000', {
  reconnectionDelay: 1000,
  reconnectionAttempts: 5
});

socket.on('connect_error', (error) => {
  console.error('接続エラー:', error.message);
});
```

### 問題2: CORSエラー

```javascript
// サーバー側でCORSを設定
const io = new Server(httpServer, {
  cors: {
    origin: "http://localhost:8080",
    methods: ["GET", "POST"]
  }
});
```

### 問題3: 予期しない切断

```javascript
// クライアント側で再接続を設定
const socket = io({
  reconnection: true,
  reconnectionDelay: 1000,
  reconnectionDelayMax: 5000,
  reconnectionAttempts: Infinity  // 無限に再接続を試みる
});

socket.on('reconnect', (attemptNumber) => {
  console.log('再接続しました。試行回数:', attemptNumber);
});
```

---

## 📝 練習課題

### 初級

1. **接続カウンター**: 現在の接続数を表示する機能を追加
2. **接続時刻表示**: 各クライアントの接続時刻をサーバーのログに記録

### 中級

3. **複数タブ対応**: 同じブラウザで複数タブを開いた時の動作を確認
4. **接続統計**: 累計接続数、現在の接続数、最大同時接続数を記録

### 上級

5. **接続制限**: 同時接続数の上限を設定し、超えた場合は接続拒否
6. **認証**: 接続時にトークンを検証し、無効な場合は切断

---

## 💡 次のステップ

基本的な接続ができるようになったら、次はイベントを使ってメッセージをやり取りしてみましょう！

👉 [Lesson 3: イベントの送受信](./03-events.md)

---

## 📚 参考リソース

- [Socket.IO Server API - Connection](https://socket.io/docs/v4/server-api/#event-connection)
- [Socket.IO Client API - connect](https://socket.io/docs/v4/client-api/#connect)
- [Handling CORS](https://socket.io/docs/v4/handling-cors/)
