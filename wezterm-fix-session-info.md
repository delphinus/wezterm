# WezTerm EmitEvent Fix - セッション情報

## 現在の状況（2025-10-24 更新）

### 問題の再発見
初期の修正では問題が完全に解決せず、multiplexing環境でpane IDのずれが依然として発生することが判明。

**テスト結果:**
1. pane 0 で CMD+e → `pane_id: 1` (ずれ: +1)
2. `wezterm cli split-pane` で分割 → pane 4
3. pane 4 で CMD+e → `pane_id: 5` (ずれ: +1)
4. 再度 split-pane → pane 7
5. pane 7 で CMD+e → `pane_id: 9` (ずれ: +2)
6. 再度 split-pane → pane 11
7. pane 11 で CMD+e → `pane_id: 14` (ずれ: +3)

**観察:** paneを分割するたびに、pane_idのずれが増加する。

### 追加調査と修正

#### 修正3: `schedule_window_event` の overlay 処理
**ファイル:** `wezterm-gui/src/termwindow/mod.rs`
**行番号:** 1575

**問題:** コメントでは「overlayを避ける」と書いていたが、実際には`get_active_pane_or_overlay()`を呼んでいた。

```diff
  let pane_id = match pane_id {
      Some(id) => id,
      None => {
          // If no pane_id specified, get the active pane from the mux.
-         // We avoid get_active_pane_or_overlay() here to ensure we get
+         // We use get_active_pane_no_overlay() here to ensure we get
          // an actual mux pane, not an overlay.
-         match self.get_active_pane_or_overlay() {
+         match self.get_active_pane_no_overlay() {
              Some(pane) => pane.pane_id(),
              None => return,
          }
      }
  };
```

#### デバッグログの追加

**WindowEvent::PerformKeyAssignment (行936):**
```rust
log::info!("WindowEvent::PerformKeyAssignment: pane_id={}", pane.pane_id());
```

**TermWindowNotif::PerformAssignment (行1137-1142):**
```rust
log::info!("TermWindowNotif::PerformAssignment: requested pane_id={}, active_pane.pane_id()={}", pane_id, active_pane.pane_id());
let pane = if active_pane.pane_id() == pane_id {
    log::info!("  -> Using active_pane (overlay or same pane)");
    active_pane
} else {
    log::info!("  -> Getting pane from mux");
    mux.get_pane(pane_id)
        .ok_or_else(|| anyhow!("pane id {} is not valid", pane_id))?
};
```

### 根本原因の分析

#### 1. KeyAssignment の実行フロー
```
CMD+e 押下
  ↓
WindowEvent::PerformKeyAssignment
  ↓ get_active_pane_or_overlay() でpaneを取得
  ↓
perform_key_assignment(&pane, &action)
  ↓
action_callback の Lua 関数が実行される
  ↓
window:perform_action(SplitPane, pane) が呼ばれる
  ↓
TermWindowNotif::PerformAssignment { pane_id: pane.pane_id(), ... }
  ↓ 再度 get_active_pane_or_overlay() を呼ぶ
  ↓
active_pane.pane_id() と requested pane_id を比較
```

#### 2. 問題の所在
`TermWindowNotif::PerformAssignment` (mod.rs:1133-1141) で、以下のロジックが使われている:

```rust
let active_pane = self.get_active_pane_or_overlay()
    .ok_or_else(|| anyhow!("there is no active pane!?"))?;
let pane = if active_pane.pane_id() == pane_id {
    active_pane  // overlay の可能性あり
} else {
    mux.get_pane(pane_id)  // mux から取得
        .ok_or_else(|| anyhow!("pane id {} is not valid", pane_id))?
};
```

このコードは、CopyMode overlay のために実装された（[#3209](https://github.com/wezterm/wezterm/issues/3209)参照）が、multiplexing環境で問題を引き起こしている可能性がある。

### 次のステップ

1. **デバッグログの確認**
   ```bash
   RUST_LOG=info ./target/release/wezterm --config-file ~/.config/wezterm/test.lua connect unix 2>&1 | grep -E "(WindowEvent::PerformKeyAssignment|TermWindowNotif::PerformAssignment)"
   ```

   ログから以下を確認:
   - `WindowEvent::PerformKeyAssignment` で取得される pane_id
   - `TermWindowNotif::PerformAssignment` の requested pane_id と active_pane.pane_id()
   - どちらのブランチ（active_pane vs mux.get_pane）が使われているか

2. **根本原因の特定**
   - multiplexing環境で`get_active_pane_or_overlay()`が間違ったpaneを返している可能性
   - client/server間の状態同期のタイミング問題の可能性
   - overlay と mux pane の ID の関係性の問題の可能性

3. **修正案の検討**
   - `TermWindowNotif::PerformAssignment`で常に`mux.get_pane(pane_id)`を優先する
   - overlayの場合の特別処理を見直す
   - multiplexing環境での状態同期を確認する

---

## 以前の修正内容

### 修正1: `EmitEvent` ハンドラー
**ファイル:** `wezterm-gui/src/termwindow/mod.rs`
**行番号:** 2847

```diff
  EmitEvent(name) => {
-     self.emit_window_event(name, None);
+     // Pass the pane_id to ensure the event handler gets the correct pane.
+     // The pane passed here might be an overlay, but overlays report the
+     // underlying pane's ID, so this should work correctly.
+     self.emit_window_event(name, Some(pane.pane_id()));
  }
```

### 修正2: `schedule_window_event` 関数
**ファイル:** `wezterm-gui/src/termwindow/mod.rs`
**行番号:** 1567-1593

```diff
  fn schedule_window_event(&mut self, name: &str, pane_id: Option<PaneId>) {
      let window = GuiWin::new(self);
-     let pane = match pane_id {
-         Some(pane_id) => Mux::get().get_pane(pane_id),
-         None => None,
-     };
-     let pane = match pane {
-         Some(pane) => pane,
-         None => match self.get_active_pane_or_overlay() {
-             Some(pane) => pane,
-             None => return,
-         },
+     let pane_id = match pane_id {
+         Some(id) => id,
+         None => {
+             // If no pane_id specified, get the active pane from the mux.
+             // We use get_active_pane_no_overlay() here to ensure we get
+             // an actual mux pane, not an overlay.
+             match self.get_active_pane_no_overlay() {
+                 Some(pane) => pane.pane_id(),
+                 None => return,
+             }
+         }
+     };
+     let pane = match Mux::get().get_pane(pane_id) {
+         Some(pane) => pane,
+         None => {
+             log::warn!(
+                 "schedule_window_event: pane {} not found in mux, event {} will not be delivered",
+                 pane_id,
+                 name
+             );
+             return;
+         }
      };
      let pane = MuxPane(pane.pane_id());
```

## ビルド済みバイナリ

**場所:** `/Users/jinnouchi.yasushi/git/github.com/wez/wezterm/target/release/`

- `wezterm` (33MB) - メインバイナリ
- `wezterm-gui` (66MB) - GUIバイナリ
- `wezterm-mux-server` (30MB) - Multiplexingサーバーバイナリ

## テスト方法

1. 現在のweztermセッションを終了
2. 新しいweztermセッションを修正済みバイナリで起動:
   ```bash
   cd /Users/jinnouchi.yasushi/git/github.com/wez/wezterm
   RUST_LOG=info ./target/release/wezterm --config-file ~/.config/wezterm/test.lua connect unix 2>&1 | tee wezterm-debug.log
   ```
3. いくつか pane を開く
4. 各 pane で `echo $WEZTERM_PANE` を実行して ID を確認（例: 0）
5. CMD+e を押す
6. 新しく開いた pane に表示される以下の値を確認:
   - `WEZTERM_PANE`: 新しい pane の ID（例: 1）
   - `pane_id`: 元の pane の ID（例: 0）← **これが正しく表示されることを確認**
7. ログファイル `wezterm-debug.log` を確認して、pane IDの流れを追跡

### 期待される結果

- **修正前**: `pane_id` の値が元の pane の ID からずれる
- **修正後**: `pane_id` の値が元の pane の ID (`$WEZTERM_PANE`) と一致

## Git差分の確認

```bash
cd /Users/jinnouchi.yasushi/git/github.com/wez/wezterm
git diff wezterm-gui/src/termwindow/mod.rs
```

## ビルド＆テスト結果

### 最新ビルド (2025-10-24)
- ✅ コンパイル成功 (`cargo build --release --bin wezterm-gui --bin wezterm-mux-server`)
- ビルド時間: 2分29秒
- デバッグログ追加済み
- すべてのバイナリが最新のソースコードでコンパイル済み

### テスト設定ファイル

**~/.config/wezterm/test.lua:**
```lua
local wezterm = require "wezterm"
local config = wezterm.config_builder()
config.unix_domains = { { name = "unix" } }
config.keys = {
  {
    key = "e",
    mods = "CMD",
    action = wezterm.action_callback(function(window, pane)
      window:perform_action(
        wezterm.action.SplitPane {
          direction = "Down",
          command = {
            args = {
              "sh",
              "-c",
              ([[
                echo "WEZTERM_PANE: $WEZTERM_PANE";
                echo "pane_id: %d";
                sleep 100;
              ]]):format(pane:pane_id()),
            },
          },
          size = { Cells = 10 },
        },
        pane
      )
    end),
  },
}
return config
```

## 関連ファイル

- `wezterm-gui/src/termwindow/mod.rs` - メインの修正ファイル
- `wezterm-gui/src/scripting/guiwin.rs` - `window:perform_action()` の実装
- `~/.config/wezterm/test.lua` - テスト用設定ファイル

## 参考情報

- Issue #3209: CopyMode overlay が pane_id をエイリアスする問題
- 関連する関数:
  - `get_active_pane_or_overlay()` - overlay を含めてアクティブな pane を取得
  - `get_active_pane_no_overlay()` - overlay を除外してアクティブな pane を取得
  - `schedule_window_event()` - window イベントをスケジュール
  - `perform_key_assignment()` - キーアサインメントを実行
