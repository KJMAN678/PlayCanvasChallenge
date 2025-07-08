# PlayCanvas 3Dモデル操作方法の変更・拡張

## 概要
現在のPlayCanvasプロジェクトでは、3Dボールの操作にWASDキーと矢印キーを使用していますが、ゲームパッド（コントローラー）対応や入力方式のカスタマイズを行うための修正が必要です。

## 現在の実装状況

### 入力システムの構成
- **キーボード入力**: WASD + 矢印キーで4方向移動
- **ゲームパッド対応**: 無効化されている（`useGamepads: false`）
- **入力処理**: Movementスクリプトで`this.app.keyboard.isPressed()`を使用

### 主要ファイル
1. `download/__settings__.js` - 入力デバイス設定
2. `download/__game-scripts.js` - Movement スクリプト（圧縮済み）
3. `download/__start__.js` - PlayCanvasアプリケーション初期化

## 必要な修正箇所

### 1. ゲームパッド対応の有効化

**ファイル**: `download/__settings__.js`
```javascript
// 現在の設定
window.INPUT_SETTINGS = {
    useKeyboard: true,
    useMouse: true,
    useGamepads: false,  // ← これをtrueに変更
    useTouch: true
};

// 修正後
window.INPUT_SETTINGS = {
    useKeyboard: true,
    useMouse: true,
    useGamepads: true,   // ← ゲームパッド対応を有効化
    useTouch: true
};
```

### 2. Movementスクリプトの拡張

**ファイル**: `download/__game-scripts.js`

現在の実装（圧縮済み）:
```javascript
Movement.prototype.update=function(t){
    const e=this.app.keyboard,
    i=e.isPressed(pc.KEY_LEFT),
    s=e.isPressed(pc.KEY_RIGHT),
    n=e.isPressed(pc.KEY_UP),
    o=e.isPressed(pc.KEY_DOWN);
    i&&this.entity.translate(-t,0,0),
    s&&this.entity.translate(t,0,0),
    n&&this.entity.translate(0,0,-t),
    o&&this.entity.translate(0,0,t)
}
```

**推奨する拡張実装**:
```javascript
Movement.prototype.update = function(deltaTime) {
    const keyboard = this.app.keyboard;
    const gamepad = this.app.gamepads ? this.app.gamepads.get(0) : null;
    
    let moveX = 0;
    let moveZ = 0;
    
    // キーボード入力
    if (keyboard.isPressed(pc.KEY_LEFT) || keyboard.isPressed(pc.KEY_A)) {
        moveX -= 1;
    }
    if (keyboard.isPressed(pc.KEY_RIGHT) || keyboard.isPressed(pc.KEY_D)) {
        moveX += 1;
    }
    if (keyboard.isPressed(pc.KEY_UP) || keyboard.isPressed(pc.KEY_W)) {
        moveZ -= 1;
    }
    if (keyboard.isPressed(pc.KEY_DOWN) || keyboard.isPressed(pc.KEY_S)) {
        moveZ += 1;
    }
    
    // ゲームパッド入力（左スティック）
    if (gamepad && gamepad.isConnected()) {
        const leftStickX = gamepad.getAxis(0); // 左スティック X軸
        const leftStickY = gamepad.getAxis(1); // 左スティック Y軸
        
        // デッドゾーン処理
        const deadzone = 0.2;
        if (Math.abs(leftStickX) > deadzone) {
            moveX += leftStickX;
        }
        if (Math.abs(leftStickY) > deadzone) {
            moveZ += leftStickY;
        }
        
        // D-pad入力
        if (gamepad.isPressed(pc.PAD_LEFT)) moveX -= 1;
        if (gamepad.isPressed(pc.PAD_RIGHT)) moveX += 1;
        if (gamepad.isPressed(pc.PAD_UP)) moveZ -= 1;
        if (gamepad.isPressed(pc.PAD_DOWN)) moveZ += 1;
    }
    
    // 移動の正規化と適用
    if (moveX !== 0 || moveZ !== 0) {
        const length = Math.sqrt(moveX * moveX + moveZ * moveZ);
        moveX = (moveX / length) * this.speed * deltaTime;
        moveZ = (moveZ / length) * this.speed * deltaTime;
        
        this.entity.translate(moveX, 0, moveZ);
    }
};
```

### 3. 入力感度の調整

Movementスクリプトに新しい属性を追加:
```javascript
Movement.attributes.add("gamepadSensitivity", {
    type: "number",
    default: 1.0,
    min: 0.1,
    max: 3.0,
    precision: 2,
    description: "ゲームパッドの入力感度"
});

Movement.attributes.add("deadzone", {
    type: "number",
    default: 0.2,
    min: 0.0,
    max: 0.5,
    precision: 2,
    description: "アナログスティックのデッドゾーン"
});
```

## 実装の優先度

### 高優先度
1. ✅ **INPUT_SETTINGS修正** - `useGamepads: true`に変更
2. ✅ **基本的なゲームパッド入力** - 左スティックとD-padでの移動

### 中優先度
3. **入力感度調整** - ゲームパッド専用の感度設定
4. **デッドゾーン処理** - アナログスティックの微細な入力を無視

### 低優先度
5. **複数コントローラー対応** - プレイヤー2以降の対応
6. **カスタムキーマッピング** - ユーザー定義のキー配置

## PlayCanvas APIリファレンス

### ゲームパッド関連API
- `this.app.gamepads.get(index)` - 指定インデックスのゲームパッドを取得
- `gamepad.isConnected()` - ゲームパッドの接続状態を確認
- `gamepad.getAxis(index)` - アナログスティックの値を取得（-1.0 ～ 1.0）
- `gamepad.isPressed(button)` - ボタンの押下状態を確認

### ゲームパッドボタン定数
- `pc.PAD_LEFT`, `pc.PAD_RIGHT`, `pc.PAD_UP`, `pc.PAD_DOWN` - D-pad
- `pc.PAD_1`, `pc.PAD_2`, `pc.PAD_3`, `pc.PAD_4` - フェイスボタン
- `pc.PAD_L1`, `pc.PAD_R1` - ショルダーボタン

## テスト方法

1. **設定変更後の確認**:
   ```bash
   cd download
   npm run dev
   ```
   http://localhost:5173/ でアプリケーションを起動

2. **ゲームパッド接続テスト**:
   - ゲームパッドをPCに接続
   - ブラウザでゲームパッドが認識されることを確認
   - 左スティックとD-padでボールが移動することを確認

3. **入力感度テスト**:
   - キーボードとゲームパッドの操作感を比較
   - 必要に応じて感度パラメータを調整

## 注意事項

- 現在の`__game-scripts.js`は圧縮されているため、修正時は読みやすい形式で書き直すことを推奨
- ゲームパッド対応はブラウザの対応状況に依存するため、主要ブラウザでのテストが必要
- PlayCanvasエンジンは既にゲームパッド機能を内蔵しているため、追加のライブラリは不要

## 関連ファイル

- `download/__settings__.js` - 入力デバイス設定
- `download/__start__.js` - アプリケーション初期化（line 252でpc.GamePads()初期化）
- `download/__game-scripts.js` - Movement, FollowCamera, Teleporter スクリプト
- `download/config.json` - アセット設定とアプリケーション設定

## 実装難易度
**難易度**: 中級 🟡

PlayCanvasの入力システムとJavaScript APIの理解が必要ですが、エンジンが提供する機能を活用するため、一から実装する必要はありません。
