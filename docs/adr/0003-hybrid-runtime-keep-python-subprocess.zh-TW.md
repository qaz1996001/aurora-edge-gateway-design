# ADR-0003：混合執行時 — 為 DSP/ML 保留 Python

**狀態**：已接受  
**決策日期**：2026-Q3  
**利益相關者**：平台工程師、DSP 團隊、資料科學團隊

---

## 背景

Aurora Edge Gateway 包括計算密集功能：
- **數位信號處理（DSP）**：感測器串流上的傅立葉轉換、濾波、頻譜分析
- **機器學習推論**：輕量異常偵測模型（TensorFlow Lite）

原始 Node.js 實施透過 `child_process.spawn` 呼叫 Python 子程序進行這些任務。問題是：用 Rust 重寫 Python 程式碼還是保留 Python？

---

## 決策

**為 DSP 與 ML 工作負載保留 Python。** Rust 透過有效率 IPC 呼叫 Python 子程序（Unix socket + bincode 序列化）。

---

## 理由

### 為何保留 Python？

**科學生態系成熟**
- `numpy`、`scipy`、`scikit-learn` 高度優化用於數值計算
- TensorFlow Lite、PyTorch Mobile 成熟用於邊緣推論
- 用 Rust 重寫所有 DSP 演算法需數週；許多已經過實戰測試
- **結果**：更快搬遷；較低引入演算法漏洞風險

**開發者生產力**
- DSP 團隊專精於 Python；不熟悉 Rust
- 將複雜 FFT 或 Kalman 濾波器移植到 Rust = 高風險
- Python 原型製作比 Rust 數值程式碼更快
- **結果**：團隊速度；數學程式碼中較少漏洞

**執行時成本可接受**
- DSP 計算攤銷 IPC 開銷（沉重計算；輕型通訊）
- IPC 往返開銷（Unix socket + bincode 序列化）相對於 DSP/ML 計算本身很小，因此跨越程序邊界預期不會成為瓶頸
- 本 ADR 並未針對 DSP/ML 執行獨立的微基準測試。在邊緣平台（4 核 ARM Cortex-A53、逐秒感測器管線、285 個樣本的量測）上，真實量測到的計算成本分佈為：JSON 反序列化 18.1 ms + RMS 計算（×3 軸）3.7 ms + 診斷計算 13.7 ms = 每週期 35.5 ms 的伺服端運算，遠低於每秒預算，顯示計算（而非 IPC）才是成本主要來源
- **結果**：在每秒延遲預算內，未觀察到相對於 Node.js 的效能衰退

### 為何不用 Rust 重寫？

**高工程量**
- 將基於 numpy 的 DSP 管線移植至 ndarray（Rust 等價物）= 數週工作
- 將 TensorFlow Lite 模型移植至 tch-rs 或 burn = 整合風險
- **成本**：2–4 週延遲；引入新漏洞可搖晃

**組織風險**
- DSP 團隊無 Rust 經驗
- 程式碼審查較慢（無一人審查複雜數值 Rust 程式碼）
- 維護負擔：未來更新更困難

**低收益**
- DSP 程式碼不在熱路徑（按需執行，非連續）
- 記憶體安全已由子程序隔離保證（Python 崩潰無法崩潰閘道）

---

## 架構

### 混合 IPC 模型

```
Rust（aeg-sensors）                  Python（子程序）
──────────────────────────────────────────────────────────
aeg-sensors 程序
│
├─ 從 MQTT 讀取感測器樣本
├─ 緩衝 1024 樣本
│
└─ [需要 FFT 分析]
   └─ 透過 Unix socket 傳送請求
      
      請求：{
        "cmd": "fft_analysis",
        "samples": [1.2, 3.4, ...],  // 1024 f64 值
        "window": "hann"
      }
      
      ↓ [Unix socket → bincode 序列化；~2 KB]
      
      Python 子程序（按需生成）
      │
      ├─ scipy.fft.fft(samples, window="hann")
      ├─ 擷取功率譜
      │
      └─ 傳送回應
      
      回應：{
        "success": true,
        "power": [10.2, 5.1, ...],  // 512 f64 值
        "peak_freq": 50.0
      }
      
      ↓ [Unix socket ← bincode；~4 KB]
      
Rust（aeg-sensors）
│
├─ 接收回應
├─ 使用功率譜進行異常偵測
└─ 更新 DB 與結果
```

### 子程序生命週期

```
┌─ aeg-sensors（Rust）
│
├─ [啟動時]
│  └─ 登錄 Python 子程序路徑：/usr/local/bin/aeg-dsp.py
│
├─ [首次 DSP 請求時]
│  └─ spawn("/usr/local/bin/aeg-dsp.py")
│  └─ 在背景保持程序活躍
│
├─ [後續 DSP 請求時]
│  └─ 透過 Unix socket 傳送請求（程序已執行）
│  └─ 無 spawn 開銷（保活）
│
├─ [錯誤或逾時時]
│  └─ 終止子程序；下次請求生成新實例
│  └─ 無級聯故障（子程序崩潰 ≠ 閘道崩潰）
│
└─ [關閉時]
   └─ 清理子程序（SIGTERM）
```

### Python 子程序介面

```python
# /usr/local/bin/aeg-dsp.py
import socket
import sys
import json
import numpy as np
from scipy import signal

def fft_analysis(samples, window='hann'):
    """使用視窗進行 FFT。"""
    windowed = samples * signal.get_window(window, len(samples))
    fft_result = np.abs(np.fft.fft(windowed))
    return fft_result[:len(samples)//2]

def anomaly_detect(recent_values, threshold=2.5):
    """簡單統計異常偵測。"""
    mean = np.mean(recent_values)
    std = np.std(recent_values)
    return (recent_values - mean) / std > threshold

# 主迴圈：從 Unix socket 讀取請求
socket_path = "/tmp/aeg-dsp.sock"
sock = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
sock.bind(socket_path)
sock.listen(1)

while True:
    conn, _ = sock.accept()
    data = conn.recv(8192)
    request = json.loads(data.decode())
    
    if request['cmd'] == 'fft_analysis':
        response = fft_analysis(
            np.array(request['samples']),
            window=request.get('window', 'hann')
        )
        conn.send(json.dumps({'success': True, 'result': response.tolist()}).encode())
    elif request['cmd'] == 'anomaly_detect':
        anomalies = anomaly_detect(np.array(request['values']))
        conn.send(json.dumps({'success': True, 'anomalies': anomalies.tolist()}).encode())
    
    conn.close()
```

---

## 結果

### 正面
1. **更快搬遷**：重用現有 Python 程式碼；無演算法重寫
2. **較低風險**：DSP 演算法已驗證；較少漏洞
3. **可維護性**：DSP 團隊可繼續維護他們的程式碼
4. **科學生態系**：無重新實施即可存取 numpy、scipy、TensorFlow Lite
5. **運營簡單性**：Python 崩潰時閘道繼續執行（隔離程序）

### 負面
1. **IPC 開銷**：每個請求因 socket 往返與序列化（bincode/JSON）而產生額外延遲；未以專門的 DSP/ML 基準測試隔離量化，但根據整體管線延遲量測（見下方「效能影響」），相對於 DSP/ML 計算本身很小
   - **緩解**：批量請求；僅用於沉重計算
2. **程序管理**：Python 子程序必須保活或有效率重生
   - **緩解**：明確子程序生命週期管理；保活策略
3. **除錯複雜性**：Rust ↔ Python 橋接除錯
   - **緩解**：兩側結構化日誌；清楚錯誤傳播
4. **依賴**：若 Python 從邊緣裝置移除，功能破壞
   - **緩解**：記錄為必要依賴；發佈 Python 執行時

### 接受的取捨
- **額外 IPC 延遲**（未以專門基準測試隔離量化，但相對於 DSP/ML 計算本身很小）換取**生態系優勢與開發速度**
- **雙執行時部署**（Rust + Python）換取**專業工作負載優勢**

---

## 效能影響

本專案並未執行逐操作（FFT、異常偵測、ML 推論）的獨立基準測試來比較 Node.js 程序內執行與 Rust + Python IPC——此專案不存在這樣的基準測試表，因此此處不提供分項數字。

我們能陳述的是，從生產邊緣管線（4 核 ARM Cortex-A53、逐秒感測器管線、285 個樣本的量測）取得的伺服端計算成本分佈（單一處理週期）：

| 階段 | 量測時間 |
|---|---|
| JSON 反序列化 | 18.1 ms |
| RMS 計算（×3 軸） | 3.7 ms |
| 診斷計算 | 13.7 ms |
| **伺服端運算總計** | **35.5 ms** |

這個 35.5 ms 的總和僅佔每秒處理預算的一小部分（端到端管線延遲：中位 55.5 ms、p95 56.4 ms——見 ADR-0005）。到 Python 子程序的 IPC 往返開銷會疊加在此之上，但相對於 DSP/ML 計算本身很小。

**結論**：IPC 往返開銷相對於 DSP/ML 計算本身很小，因此未觀察到程序邊界成為瓶頸。這是根據整體管線延遲得出的質化結論，並非獨立的 IPC 基準測試結果。

---

## 監控與除錯

### 子程序健康
```bash
# 檢查 Python 子程序是否執行
ps aux | grep aeg-dsp.py

# 監控子程序 CPU/記憶體
watch -n 1 'ps -o %cpu,%mem,cmd | grep aeg-dsp'

# 查看子程序日誌（若有單獨日誌檔）
tail -f /var/log/aeg-dsp.log
```

### IPC 效能
```bash
# 新增追蹤以測量 IPC 往返時間
RUST_LOG=aeg_sensors=debug aeg-sensors

# 範例除錯輸出（僅示意日誌格式，非量測基準）：
# [DEBUG] aeg_sensors：傳送 FFT 請求（1024 樣本）
# [DEBUG] aeg_sensors：Python 回應於 <N>ms 內收到
```

---

**狀態**：✅ **已接受**（2026-Q3）  
**批准者**：平台團隊、DSP 團隊  
**實施**：Phase A（第 4 週前 IPC 橋接）
