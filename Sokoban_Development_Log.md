# 🛠️ Defold 獨立遊戲開發實戰筆記：2D 倉庫番（Sokoban）

這是一份專為從復古街機硬體思維跨入 2D 遊戲開發的獨立開發生態筆記。記錄了如何利用 Defold 引擎，在死守硬體上限（Fixed Budget）的主機/行動端思維下，完成一款具備「數據與畫面同步、穿牆防禦、聲光完美閉環」的商業街機級 2D 倉庫番遊戲。

---

## 💾 1. 硬體血緣與開發前置思維
在實戰開始前，我們透過 SEGA NAOMI（32MB VRAM）與 Sammy Atomiswave（16MB VRAM，本質上是卡帶版 Dreamcast）的歷史與民間轉換版（`.bin` 格式）學到了核心主機開發哲學：
* **死守上限（Fixed Budget）**：主機/行動端開發不同於電腦（PC），記憶體上限邊界被完全鎖死，多 1KB 都有可能引發硬體直接死機。
* **杜絕 GC 停頓（Garbage Collection）**：在每秒 60 幀（1 幀僅 16.6 毫秒）的遊戲中，運行時的垃圾回收會造成致命的畫面卡頓（Stuttering），這也是現代開發者推崇 Rust 引擎或高度優化之輕量引擎（如 Defold + LuaJIT）的原因。

### ⚙️ 專案基礎環境設定
*   **引擎版本**：Defold 1.12.4 (Empty Project 範本開始)
*   **畫面解析度設定**：正方形復古街機比例（於 `game.project` 中設定 `Display` 寬高為 `640 × 640`）。

---

## 🎨 2. 美術材質與圖層優化（Texture & Tilemap）

### 📦 圖片記憶體（VRAM）壓榨定律
*   **PNG 壓縮格式誤區**：PNG 是硬碟上的壓縮格式，引擎渲染時必須在 VRAM 中完全解壓縮成像素點陣。一張 2048x2048 的 PNG 固定消耗 16MB VRAM。
*   **2 的 N 次方定律 (Power of 2)**：手機與主機 GPU 處理 `128x128`、`512x512`、`1024x1024` 的圖集效率最高，非標準尺寸會導致 GPU 底層強行開闢更大空間，白白浪費記憶體。
*   **行動端終極壓縮**：Android 導出首選 `ETC2` 格式，iOS 首選 `ASTC` 格式，可將 VRAM 壓低至原本的 25% 且不失真。

### 🖌️ 手工拼製精靈圖集（Spritesheet）
由於 Defold 網格地圖（Tilemap）底層規範，無法直接選取 `.atlas` 作為磁磚源。我們採用 Mac 內建預覽程式的「複製貼上黑魔法」，將 Kenney 5 張基礎素材（`128x128`）手工拼接成一張 `640x128` 的標準大圖 `sokoban_tiles.png`：
1.  **牆壁 (Wall)** ── 2. **地板 (Ground)** ── 3. **箱子 (Box)** ── 4. **目標金幣 (Target)** ── 5. **主角 (Player)**

### 🗺️ 雙圖層分離技術（Layer Management）
在建立 `level.tilemap` 時，必須將「背景層」與「動態物件層」完美隔離，防止物件移動後漏出底層全黑深淵：
*   **`layer1`（背景層）**：負責用滑鼠刷出圍牆（Wall）與鋪滿內側的地板（Ground）。金幣（Target）為了能被箱子在上方重疊碾壓，也必須在此層繪製。
*   **`layer2`（動態物件層）**：負責擺放起點的箱子（Box）與主角（Player）。
*   *Defold 圖磚清除手勢*：選中圖層後，在黑色畫布上按住 **`Shift + 滑鼠右鍵`** 即可作為全能橡皮擦。

---

## 🧠 3. 輸入連線與核心邏輯控制（Input & Game Loop）

### 🎛️ 建立輸入綁定（Input Bindings）
在 `main` 資料夾下新建 `game.input_binding`（並於 `game.project -> Input -> Input Bindings` 完成連線），打通外設神經系統：
*   `Key Triggers`：將 `key_up/down/left/right` 映射為對應的 Action 代碼 `up/down/left/right`。

### 📄 終極大腦邏輯程式碼：`logic.script`
此腳本掛載於主舞台 `main.collection` 的 `map` 遊戲物件下。實作了**「主角與箱子穿牆防禦 check」**與**「差一錯誤（Off-by-one error）微調」**。
*   **碰撞格子邊界範圍**：左上角 `(3,8)`，右下角 `(8,4)`。
*   **核心修煉：邊界防禦碼（Boundary Check）**：當箱子要前進時，前方的探路針會摸到牆壁區段 `next_x >= 8`，為了放寬讓箱子完美進站至 `(7, 5)` 金幣網格內，邊界攔截碼必須精準微調放寬至 `9`（小於 9 即可通行）。

```lua
local TILE_SIZE = 128

local TILE_WALL   = 1
local TILE_GROUND = 2
local TILE_BOX    = 3
local TILE_TARGET = 4
local TILE_PLAYER = 5

-- 🏆 巡邏金幣的精準網格座標 (根據 level.tilemap 的實體 Cell 座標)
local TARGET_POSITIONS = {
    { x = 7, y = 5 }
}

function init(self)
    msg.post(".", "acquire_input_focus")
    -- 精準校正主角開機時在 Tilemap 的真實格子座標
    self.player_x = 4
    self.player_y = 7
    self.is_game_over = false
    
    -- 🚀 音樂絕對路徑發射碼
    sound.play("main:/map#bgm")
end

-- 🏆 核心獲勝判定巡邏函式
local function check_win_condition()
    local url = msg.url("#level")
    local all_targets_covered = true

    for _, pos in ipairs(TARGET_POSITIONS) do
        local layer2_tile = tilemap.get_tile(url, "layer2", pos.x, pos.y)
        if layer2_tile ~= TILE_BOX then
            all_targets_covered = false
            break
        end
    end

    if all_targets_covered then
        -- 精準投遞勝利命令到 UI 視窗組件
        msg.post("#ui", "show_win")
    end
end

local function try_move(self, dx, dy)
    if self.is_game_over then return end

    local target_x = self.player_x + dx
    local target_y = self.player_y + dy
    local next_x = target_x + dx
    local next_y = target_y + dy

    -- 🛡️ 主角邊界攔截 (3-8, 4-8)
    if target_x <= 3 or target_x >= 8 or target_y <= 4 or target_y >= 8 then
        return
    end

    local layer1_tile = tilemap.get_tile("#level", "layer1", target_x, target_y)
    local layer2_tile = tilemap.get_tile("#level", "layer2", target_x, target_y)

    if layer1_tile == TILE_WALL or layer2_tile == TILE_WALL then
        return
    end

    -- 🛡️ 推箱子邏輯與邊界放寬限制
    if layer2_tile == TILE_BOX then
        -- 差一錯誤修正：允許箱子踩到 7 (目的地 next_x 摸到 8，小於 9 即可通行)
        if next_x <= 3 or next_x >= 9 or next_y <= 3 or next_y >= 9 then
            return
        end

        local next_layer2 = tilemap.get_tile("#level", "layer2", next_x, next_y)
        if next_layer2 == 0 then
            tilemap.set_tile("#level", "layer2", target_x, target_y, 0)
            tilemap.set_tile("#level", "layer2", next_x, next_y, TILE_BOX)
        else
            return
        end
    end

    -- 移動更新主角
    tilemap.set_tile("#level", "layer2", self.player_x, self.player_y, 0)
    self.player_x = target_x
    self.player_y = target_y
    tilemap.set_tile("#level", "layer2", self.player_x, self.player_y, TILE_PLAYER)

    -- 👣 主角移動完畢，發送內部通訊訊號觸發檢查
    msg.post("#", "check_rules")
end

function on_input(self, action_id, action)
    if action.pressed then
        if action_id == hash("up") then try_move(self, 0, 1)
        elseif action_id == hash("down") then try_move(self, 0, -1)
        elseif action_id == hash("left") then try_move(self, -1, 0)
        elseif action_id == hash("right") then try_move(self, 1, 0)
        end
    end
end

function on_message(self, message_id, message, sender)
    if message_id == hash("check_rules") then
        check_win_condition()
    elseif message_id == hash("game_set") then
        self.is_game_over = true
        -- 🤫 獲勝時切斷背景音樂
        sound.stop("#bgm")
    end
end
```

---

## 🎆 4. 獲勝 UI 與全自動通訊（GUI & Message Loop）

### 🖥️ 建立過關畫面 UI（`ui.gui`）
1.  在 `Nodes` 右鍵 `Add > Text`。設定 `Id` 為 **`win_text`**，文字內容為 `"YOU WIN!"`，座標置中於 `(320, 320)`，縮放比例（Scale）放大。
2.  **`Enabled` 取消勾選**：核心安全機制，開機時必須隱形，直到大腦巡邏過關才現身。
3.  **字型引用硬規則**：Defold 的 GUI 屬於獨立沙盒，必須在 Outline 的 `Fonts` 資料夾右鍵 `Add > Fonts` 引進系統預設字型 `system_font.font`，文字節點的 `Font` 屬性才能順利指派。

### 📡 異步訊息調度中心：`ui.gui_script`
透過相對路徑與 `sender` 指標，完成 UI 與遊戲大腦腳本的「解耦（Decoupling）通訊」：

```lua
function on_message(self, message_id, message, sender)
    -- 當收到邏輯大腦傳來的勝利快遞時
    if message_id == hash("show_win") then
        -- 讓大字體浮現
        local win_node = gui.get_node("win_text")
        gui.set_enabled(win_node, true)
        
        -- 🚀 藉由傳送者指標（sender）回傳鎖死訊號，通知 logic 腳本將遊戲物理定格
        msg.post(sender, "game_set")
    end
end
```

---

## 🎶 5. ffmpeg 工業級轉碼與背景音樂（Audio Pipeline）

### 🚨 遭遇錯誤：`INVALID_DATA / Failed to detect sound type`
*   **成因**：即使副檔名是 `.ogg`，若內部實際編碼為 MP3 偽裝、或是 Defold 不支援的變體 Opus 編碼，會被引擎拒絕播放。Defold 唯一指定支援 **標準 Ogg Vorbis（16-bit, 44100Hz）**。

### 🛠️ ffmpeg 終端機系統原生轉碼黑魔法
若本地 `ffmpeg` 未內嵌 `libvorbis` 編碼包（跳出 `Unknown encoder`），可直接調用 Mac 系統內建的原生音訊引擎（`-c:a vorbis`），一鍵輸出完美合規的音訊：

```bash
ffmpeg -i bgm.ogg -c:a vorbis -ar 44100 -q:a 5 -y perfect_bgm.ogg
```
將輸出的 `perfect_bgm.ogg` 重新拉入 Defold 改名為 `bgm.ogg`，並在 `main.collection` 的 Sound 組件中**勾選 `Looping`**，即達成全聲光效果之街機倉庫番完美完工！

---
### 💡 日後擴充備忘錄
*   **新增金幣/箱子**：前往 `logic.script` 頂部的 `TARGET_POSITIONS` 清單中往下加一行大括號座標。
*   **視窗焦點安全機制**：若按 `Cmd + B` 開機發現沒音樂或不能動，用滑鼠點擊一下遊戲視窗中央以獲取 Window Focus 即可。
