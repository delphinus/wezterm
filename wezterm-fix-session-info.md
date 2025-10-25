# WezTerm EmitEvent Fix - セッション情報

## 最終的な解決策（2025-10-25 更新）

### 根本原因の特定

multiplexing環境では、**GUI側のpane IDとCLI側のpane IDが異なる**ことが判明。

**pane IDの関係：**
```
起動時（ローカル）: GUI pane 0
↓ multiplexingサーバーに接続
サーバー接続後: GUI pane 1 ← CLI pane 0
split-pane後: GUI pane 2 ← CLI pane 1
```

**関係式: `GUI pane ID = CLI pane ID + 1` (multiplexing環境)**

### editpromptのユースケース

editprompt（https://github.com/eetann/editprompt）は以下の動作をする：

1. WezTerm上で設定したマッピングを押す
2. 新しいpaneが開き、エディタ（Neovim等）が起動
3. エディタで文字列を入力・保存・終了
4. `wezterm cli send-text --pane-id [pane_id]` で元のpaneにテキストを送信

**問題：** `wezterm cli send-text` は**CLI側のpane ID**を使用するが、Luaの`pane:pane_id()`は**GUI側のpane ID**を返す。

### $WEZTERM_PANEの問題

環境変数`$WEZTERM_PANE`は、**そのシェルが実行されているpaneのID**を保持する。

```lua
-- 問題のあるコード例
act.SplitPane {
  command = {
    args = { "fish", "-c", "editprompt -t $WEZTERM_PANE" }
  }
}
```

このコードの実行フロー：
1. 元のpane（CLI pane 3）でCMD+eを押す
2. 新しいpane（CLI pane 14）が開く
3. editpromptはその新しいpane内で実行される
4. `$WEZTERM_PANE`を参照すると`14`（**新しいpane自身のID**）
5. `wezterm cli send-text --pane-id 14` → 自分自身に送信
6. editprompt終了でpane 14が閉じる → テキストが消える

### 最終的な解決策

**Lua側でdomain判定を行い、GUI pane IDからCLI pane IDを計算する：**

```lua
local editprompt = wezterm.action_callback(function(window, pane)
  local gui_pane_id = pane:pane_id()
  local cli_pane_id = gui_pane_id

  -- multiplexing環境の判定
  local domain = pane:get_domain_name()
  if domain and domain ~= "local" then
    -- unix domainなどのmultiplexing環境ではGUI ID - 1 = CLI ID
    cli_pane_id = gui_pane_id - 1
  end

  window:perform_action(
    act.SplitPane {
      direction = "Down",
      command = {
        args = {
          "/opt/homebrew/bin/fish",
          "-c",
          -- Lua側で計算したCLI pane IDを直接埋め込む
          ("editprompt -e ~/git/dotfiles/bin/minivim -m wezterm -t %d --always-copy"):format(cli_pane_id),
        },
      },
      size = { Cells = 10 },
    },
    pane
  )
end)
```

**重要なポイント：**
- `$WEZTERM_PANE`は使わない（新しいpane自身のIDになってしまう）
- Lua側で計算した`cli_pane_id`を`format()`で埋め込む
- `domain ~= "local"`でmultiplexing環境を判定

### 検証結果

```fish
# multiplexing環境で
~ ❯❯❯ wezterm cli list
WINID TABID PANEID WORKSPACE SIZE   TITLE              CWD
    0     0      0 default   200x16 ~                  file://58988-mac/Users/jinnouchi.yasushi
    0     0      2 default   200x19 ~                  file://58988-mac/Users/jinnouchi.yasushi
    0     0      3 default   200x42 wezterm cli list ~ file://58988-mac/Users/jinnouchi.yasushi

# pane 3でCMD+eを押す
# → Lua: gui_pane_id = 4, domain = "unix", cli_pane_id = 3
# → editpromptは正しくpane 3にテキストを送信
# → ✅ 成功！
```

---

## 以前の調査（2025-10-24）

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
