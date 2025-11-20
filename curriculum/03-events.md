# Lesson 3: イベントの送受信

## 🎯 学習目標

- カスタムイベントを作成・送信する
- イベントリスナーを登録する
- データをイベントと一緒に送信する
- イベントのAcknowledgement（確認応答）を実装する

---

## 📋 メリット

✅ **柔軟なメッセージング**: 任意の名前のイベントを定義できる
✅ **型安全なデータ**: JSONシリアライズ可能なあらゆるデータを送信可能
✅ **双方向通信**: クライアント・サーバー両方から送信できる
✅ **確認応答**: メッセージが届いたことを確認できる
✅ **複数引数**: 1つのイベントで複数のデータを送信可能

---

## ⚠️ デメリット

❌ **イベント名の管理**: イベント名のタイポでバグが発生しやすい
❌ **データサイズ制限**: 大きなデータの送信には不向き
❌ **順序保証**: 複数イベントの到着順序は保証されない（通常は順序通り）
❌ **エラーハンドリング**: イベントリスナー内のエラーが伝播しない

---

## ⚙️ 技術的原理

### イベントの送信フロー

```
クライアント                                    サーバー
    │                                              │
    │  socket.emit('eventName', data)             │
    ├─────────────────────────────────────────────►│
    │                                              │
    │  1. データをJSONにシリアライズ                │
    │  2. Socket.IOプロトコルパケットを作成         │
    │  3. Engine.IOでエンコード                    │
    │  4. WebSocketで送信                          │
    │                                              │
    │                          5. WebSocketで受信  │
    │                    6. Engine.IOでデコード    │
    │          7. Socket.IOプロトコルをパース       │
    │                  8. JSONをデシリアライズ      │
    │        9. イベントリスナーを呼び出し          │
    │                    socket.on('eventName')    │
    │                                              │
```

### パケット構造

Socket.IOのイベントは以下の形式でエンコードされます：

```
42["eventName",{"key":"value"}]
└┬┘└────────┬────────────────┘
 │          └─ JSONペイロード
 └─ パケットタイプ (4=EVENT, 2=MESSAGE)

Acknowledgementありの場合：
421["eventName",{"key":"value"}]
└┬┘└┬┘└────────┬──────────────┘
 │  │          └─ JSONペイロード
 │  └─ ACK ID (応答用の識別子)
 └─ パケットタイプ
```

### メモリとパフォーマンス

- **イベントバッファ**: 切断中に送信されたイベントはバッファに保存される
- **シリアライゼーション**: JSON.stringify/parseのコストがかかる
- **バイナリデータ**: ArrayBuffer、Blobも送信可能（自動検出）

---

## 💼 ユースケース

### 適している場面

1. **チャットメッセージ**: `message` イベントでテキストを送信
2. **ステータス更新**: `statusChange` イベントでユーザーの状態を通知
3. **通知**: `notification` イベントでアラートを送信
4. **ゲームアクション**: `playerMove` イベントで位置を送信

### 注意が必要な場面

1. **大容量ファイル**: イベントではなくHTTP APIを使用すべき
2. **高頻度の送信**: 秒間数百回以上のイベント送信（スロットリング必要）
3. **機密データ**: 暗号化なしで重要データを送信（暗号化推奨）

---

## 💻 実装例

### ステップ1: プロジェクトのセットアップ

```bash
mkdir socket-events
cd socket-events
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

io.on('connection', (socket) => {
  console.log(`✅ 接続: ${socket.id}`);

  // ===== 基本的なイベント受信 =====

  // 単純なメッセージイベント
  socket.on('message', (data) => {
    console.log('📨 メッセージ受信:', data);
  });

  // 複数の引数を受け取る
  socket.on('chat', (username, message, timestamp) => {
    console.log(`💬 [${timestamp}] ${username}: ${message}`);
  });

  // オブジェクトを受け取る
  socket.on('userAction', (action) => {
    console.log('🎬 ユーザーアクション:', {
      type: action.type,
      user: action.userId,
      data: action.data
    });
  });

  // ===== Acknowledgement（確認応答）=====

  // ACKコールバックを使用
  socket.on('requestData', (query, callback) => {
    console.log('🔍 データリクエスト:', query);

    // データベースからデータを取得する想定
    const responseData = {
      success: true,
      data: {
        id: 1,
        name: 'サンプルデータ',
        timestamp: new Date().toISOString()
      }
    };

    // コールバックで応答を返す
    callback(responseData);
  });

  // エラーハンドリング付きACK
  socket.on('saveData', (data, callback) => {
    console.log('💾 データ保存リクエスト:', data);

    try {
      // データ検証
      if (!data.name || !data.value) {
        callback({
          success: false,
          error: '必須フィールドが不足しています'
        });
        return;
      }

      // データ保存処理（実際はDBに保存）
      console.log('✅ データ保存成功');

      callback({
        success: true,
        savedId: Math.floor(Math.random() * 1000)
      });
    } catch (error) {
      callback({
        success: false,
        error: error.message
      });
    }
  });

  // ===== サーバーからイベント送信 =====

  // 接続クライアントにウェルカムメッセージ
  socket.emit('welcome', {
    message: 'サーバーへようこそ！',
    serverTime: new Date().toISOString(),
    socketId: socket.id
  });

  // 5秒後にサーバーからイベント送信
  setTimeout(() => {
    socket.emit('serverMessage', {
      type: 'info',
      text: 'これはサーバーから送信されたメッセージです',
      timestamp: Date.now()
    });
  }, 5000);

  // ===== タイムアウト付きACK =====

  // クライアントにデータを要求し、タイムアウトを設定
  socket.timeout(5000).emit('getData', (err, response) => {
    if (err) {
      console.log('⏰ タイムアウト: クライアントから応答がありません');
    } else {
      console.log('📥 クライアントからのデータ:', response);
    }
  });

  socket.on('disconnect', () => {
    console.log(`❌ 切断: ${socket.id}`);
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
  <title>Socket.IO - イベントの送受信</title>
  <style>
    body {
      font-family: Arial, sans-serif;
      max-width: 900px;
      margin: 20px auto;
      padding: 20px;
      background-color: #f5f5f5;
    }
    .container {
      background-color: white;
      border-radius: 8px;
      padding: 20px;
      margin-bottom: 20px;
      box-shadow: 0 2px 5px rgba(0,0,0,0.1);
    }
    h1 {
      color: #333;
      border-bottom: 3px solid #4CAF50;
      padding-bottom: 10px;
    }
    h2 {
      color: #666;
      font-size: 18px;
      margin-top: 20px;
    }
    button {
      background-color: #4CAF50;
      color: white;
      padding: 10px 20px;
      border: none;
      border-radius: 5px;
      cursor: pointer;
      margin: 5px;
      font-size: 14px;
    }
    button:hover {
      background-color: #45a049;
    }
    input, textarea {
      width: 100%;
      padding: 10px;
      margin: 5px 0;
      border: 1px solid #ddd;
      border-radius: 4px;
      box-sizing: border-box;
    }
    #log {
      background-color: #f8f9fa;
      border: 1px solid #ddd;
      border-radius: 5px;
      padding: 15px;
      max-height: 400px;
      overflow-y: auto;
      font-family: monospace;
      font-size: 13px;
    }
    .log-entry {
      margin: 5px 0;
      padding: 8px;
      border-left: 3px solid #4CAF50;
      padding-left: 10px;
      background-color: white;
    }
    .log-entry.received {
      border-left-color: #2196F3;
    }
    .log-entry.sent {
      border-left-color: #FF9800;
    }
    .log-entry.error {
      border-left-color: #f44336;
      background-color: #ffebee;
    }
    .section {
      margin: 20px 0;
      padding: 15px;
      background-color: #f8f9fa;
      border-radius: 5px;
    }
  </style>
</head>
<body>
  <div class="container">
    <h1>📡 Socket.IO - イベントの送受信</h1>

    <!-- 基本的なイベント送信 -->
    <div class="section">
      <h2>1️⃣ 基本的なイベント送信</h2>
      <input type="text" id="simpleMessage" placeholder="メッセージを入力">
      <button onclick="sendSimpleMessage()">シンプルメッセージ送信</button>
    </div>

    <!-- 複数引数のイベント -->
    <div class="section">
      <h2>2️⃣ 複数引数のイベント</h2>
      <input type="text" id="username" placeholder="ユーザー名" value="太郎">
      <input type="text" id="chatMessage" placeholder="チャットメッセージ">
      <button onclick="sendChatMessage()">チャットメッセージ送信</button>
    </div>

    <!-- オブジェクトを送信 -->
    <div class="section">
      <h2>3️⃣ オブジェクトを送信</h2>
      <select id="actionType">
        <option value="click">クリック</option>
        <option value="hover">ホバー</option>
        <option value="scroll">スクロール</option>
      </select>
      <button onclick="sendUserAction()">アクション送信</button>
    </div>

    <!-- Acknowledgement（確認応答） -->
    <div class="section">
      <h2>4️⃣ Acknowledgement（確認応答）</h2>
      <button onclick="requestData()">データリクエスト（ACK付き）</button>
      <button onclick="saveData()">データ保存（ACK付き）</button>
    </div>

    <!-- ログ -->
    <div class="container">
      <h2>📋 イベントログ</h2>
      <button onclick="clearLog()" style="background-color: #f44336;">ログクリア</button>
      <div id="log"></div>
    </div>
  </div>

  <script src="/socket.io/socket.io.js"></script>
  <script>
    const socket = io();

    // ===== ログ表示機能 =====
    function addLog(message, type = 'info') {
      const log = document.getElementById('log');
      const entry = document.createElement('div');
      entry.className = `log-entry ${type}`;
      const timestamp = new Date().toLocaleTimeString();
      entry.textContent = `[${timestamp}] ${message}`;
      log.appendChild(entry);
      log.scrollTop = log.scrollHeight;
    }

    function clearLog() {
      document.getElementById('log').innerHTML = '';
    }

    // ===== イベント受信 =====

    socket.on('connect', () => {
      addLog('✅ サーバーに接続しました', 'received');
    });

    socket.on('welcome', (data) => {
      addLog(`📨 受信: welcome`, 'received');
      addLog(`   メッセージ: ${data.message}`, 'received');
      addLog(`   サーバー時刻: ${data.serverTime}`, 'received');
    });

    socket.on('serverMessage', (data) => {
      addLog(`📨 受信: serverMessage`, 'received');
      addLog(`   タイプ: ${data.type}`, 'received');
      addLog(`   テキスト: ${data.text}`, 'received');
    });

    // サーバーからのデータリクエストに応答
    socket.on('getData', (callback) => {
      addLog('📨 受信: getData（サーバーがデータを要求）', 'received');

      // クライアント側のデータを返す
      const clientData = {
        userAgent: navigator.userAgent,
        language: navigator.language,
        screenSize: `${screen.width}x${screen.height}`,
        timestamp: new Date().toISOString()
      };

      callback(clientData);
      addLog('📤 送信: クライアントデータを返却', 'sent');
    });

    // ===== イベント送信関数 =====

    // 1. シンプルなメッセージ送信
    function sendSimpleMessage() {
      const message = document.getElementById('simpleMessage').value;
      if (!message) {
        alert('メッセージを入力してください');
        return;
      }

      socket.emit('message', message);
      addLog(`📤 送信: message - "${message}"`, 'sent');
      document.getElementById('simpleMessage').value = '';
    }

    // 2. 複数引数で送信
    function sendChatMessage() {
      const username = document.getElementById('username').value;
      const message = document.getElementById('chatMessage').value;

      if (!message) {
        alert('メッセージを入力してください');
        return;
      }

      const timestamp = new Date().toLocaleTimeString();
      socket.emit('chat', username, message, timestamp);

      addLog(`📤 送信: chat`, 'sent');
      addLog(`   ユーザー: ${username}`, 'sent');
      addLog(`   メッセージ: ${message}`, 'sent');

      document.getElementById('chatMessage').value = '';
    }

    // 3. オブジェクトを送信
    function sendUserAction() {
      const actionType = document.getElementById('actionType').value;

      const action = {
        type: actionType,
        userId: socket.id,
        data: {
          x: Math.floor(Math.random() * 100),
          y: Math.floor(Math.random() * 100)
        },
        timestamp: Date.now()
      };

      socket.emit('userAction', action);
      addLog(`📤 送信: userAction - ${actionType}`, 'sent');
    }

    // 4. Acknowledgement付きデータリクエスト
    function requestData() {
      const query = {
        type: 'user',
        id: 1
      };

      addLog('📤 送信: requestData（ACK付き）', 'sent');

      socket.emit('requestData', query, (response) => {
        // サーバーからの応答を受信
        addLog('📨 ACK受信: requestData', 'received');
        addLog(`   成功: ${response.success}`, 'received');
        addLog(`   データ: ${JSON.stringify(response.data)}`, 'received');
      });
    }

    // 5. データ保存（エラーハンドリング付き）
    function saveData() {
      const data = {
        name: 'テストデータ',
        value: Math.floor(Math.random() * 100),
        timestamp: new Date().toISOString()
      };

      addLog('📤 送信: saveData（ACK付き）', 'sent');

      socket.emit('saveData', data, (response) => {
        if (response.success) {
          addLog('✅ ACK受信: 保存成功', 'received');
          addLog(`   保存ID: ${response.savedId}`, 'received');
        } else {
          addLog('❌ ACK受信: 保存失敗', 'error');
          addLog(`   エラー: ${response.error}`, 'error');
        }
      });
    }

    // Enterキーでメッセージ送信
    document.getElementById('simpleMessage').addEventListener('keypress', (e) => {
      if (e.key === 'Enter') sendSimpleMessage();
    });

    document.getElementById('chatMessage').addEventListener('keypress', (e) => {
      if (e.key === 'Enter') sendChatMessage();
    });
  </script>
</body>
</html>
```

---

## 🔍 コードの詳細解説

### イベントの送信パターン

```javascript
// 1. データなしでイベント送信
socket.emit('eventName');

// 2. 単一のデータを送信
socket.emit('message', 'Hello');

// 3. 複数の引数を送信
socket.emit('chat', 'username', 'message', timestamp);

// 4. オブジェクトを送信
socket.emit('data', { key: 'value', number: 123 });

// 5. 配列を送信
socket.emit('list', [1, 2, 3, 4, 5]);

// 6. Acknowledgement（コールバック）付き
socket.emit('request', data, (response) => {
  console.log('サーバーからの応答:', response);
});

// 7. タイムアウト付き送信（サーバー側のみ）
socket.timeout(5000).emit('event', (err, response) => {
  if (err) {
    // タイムアウト
  } else {
    // 正常応答
  }
});
```

### イベントの受信パターン

```javascript
// 1. イベントをリッスン
socket.on('eventName', (data) => {
  console.log(data);
});

// 2. 複数引数を受け取る
socket.on('chat', (username, message, timestamp) => {
  console.log(`${username}: ${message}`);
});

// 3. Acknowledgementで応答
socket.on('request', (data, callback) => {
  // 処理を実行
  const result = processData(data);
  // コールバックで応答
  callback(result);
});

// 4. 一度だけ実行されるリスナー
socket.once('oneTimeEvent', (data) => {
  console.log('このイベントは一度だけ処理されます');
});

// 5. リスナーを削除
socket.off('eventName');  // すべてのリスナーを削除
socket.off('eventName', specificHandler);  // 特定のリスナーのみ削除
```

### バイナリデータの送信

```javascript
// ArrayBuffer
const buffer = new ArrayBuffer(4);
socket.emit('binary', buffer);

// Blob（クライアント側）
const blob = new Blob(['Hello'], { type: 'text/plain' });
socket.emit('file', blob);

// Buffer（サーバー側Node.js）
const buf = Buffer.from('Hello');
socket.emit('buffer', buf);
```

---

## 🧪 動作確認

### テスト手順

1. サーバー起動: `node server.js`
2. ブラウザで `http://localhost:3000` を開く
3. 各ボタンをクリックしてイベントを送信
4. ログで送受信を確認
5. サーバーのコンソールでも受信を確認

### 期待される動作

```
// クライアント側
[14:30:15] ✅ サーバーに接続しました
[14:30:15] 📨 受信: welcome
[14:30:15]    メッセージ: サーバーへようこそ！
[14:30:20] 📨 受信: serverMessage
[14:30:20]    テキスト: これはサーバーから送信されたメッセージです

// サーバー側
✅ 接続: Xy3kj2nW_4XnN4XWAAAB
📨 メッセージ受信: こんにちは
💬 [14:30:25] 太郎: こんにちは
🔍 データリクエスト: { type: 'user', id: 1 }
```

---

## 🐛 よくある問題

### 問題1: イベントが受信されない

```javascript
// ❌ 悪い例: イベント名のタイポ
socket.emit('messge', 'Hello');  // typo!
socket.on('message', (data) => {  // 受信できない
  console.log(data);
});

// ✅ 良い例: 定数で管理
const EVENTS = {
  MESSAGE: 'message',
  CHAT: 'chat',
  USER_ACTION: 'userAction'
};

socket.emit(EVENTS.MESSAGE, 'Hello');
socket.on(EVENTS.MESSAGE, (data) => {
  console.log(data);
});
```

### 問題2: Acknowledgementのタイムアウト

```javascript
// クライアント側: タイムアウト処理を追加
const timeout = setTimeout(() => {
  console.log('⏰ タイムアウト');
}, 5000);

socket.emit('request', data, (response) => {
  clearTimeout(timeout);
  console.log('応答受信:', response);
});
```

### 問題3: イベントリスナーの重複登録

```javascript
// ❌ 悪い例: 何度も呼ばれると重複登録
function setupListeners() {
  socket.on('message', handleMessage);  // 重複！
}

// ✅ 良い例: 既存のリスナーを削除してから登録
function setupListeners() {
  socket.off('message', handleMessage);  // 削除
  socket.on('message', handleMessage);   // 登録
}

// または once() を使用
socket.once('message', handleMessage);
```

---

## 📝 練習課題

### 初級
1. **カウンター**: ボタンクリックでカウントを増やし、サーバーに送信
2. **エコーサーバー**: サーバーが受信したメッセージをそのまま返す

### 中級
3. **タイピングインジケーター**: 入力中であることを他のユーザーに通知
4. **ステータス更新**: オンライン/オフライン/離席のステータスを送信

### 上級
5. **プログレス追跡**: 長時間処理の進捗をリアルタイムで送信
6. **バッチ送信**: 複数のイベントをまとめて送信する機能

---

## 💡 次のステップ

イベントの送受信ができるようになったら、次は複数のクライアントに同時送信する「ブロードキャスト」を学びましょう！

👉 [Lesson 4: ブロードキャスト](./04-broadcast.md)

---

## 📚 参考リソース

- [Socket.IO - Emitting events](https://socket.io/docs/v4/emitting-events/)
- [Socket.IO - Listening to events](https://socket.io/docs/v4/listening-to-events/)
- [Socket.IO - Acknowledgements](https://socket.io/docs/v4/emitting-events/#acknowledgements)
