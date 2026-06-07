# Day 1 學習文件：BMC 基礎概念

## 今天的學習目標

讀完這份文件後，你應該能用自己的話回答：

1. BMC 是什麼？
2. BMC 和一般 MCU firmware 有什麼不同？
3. Host OS 當機時，BMC 為什麼仍可工作？
4. IPMI 與 Redfish 是什麼？它們有什麼差異？

今天不需要理解 BMC 的所有實作細節。重點是先建立正確的系統概念。

---

## 1. 先理解一台伺服器

一般使用者看到的伺服器，主要由以下部分組成：

- CPU、記憶體、硬碟、網路卡等硬體
- 在主要 CPU 上執行的作業系統，例如 Linux 或 Windows Server
- 提供網站、資料庫等功能的應用程式

這個主要作業系統通常稱為 **Host OS**。

如果 Host OS 當機、正在重新啟動，甚至還沒有安裝，管理員仍然需要知道伺服器發生了什麼事，也可能需要從遠端重新開機。這就是 BMC 存在的重要原因。

---

## 2. BMC 是什麼？

BMC 是 **Baseboard Management Controller** 的縮寫，可以先把它理解成：

> 位於伺服器主機板上、專門負責管理與監控伺服器的獨立小型電腦。

BMC 通常有自己的：

- 處理器或 SoC
- 記憶體
- Firmware 或 Embedded Linux
- 網路介面
- 電源來源
- 與主機板硬體溝通的介面

BMC 常見的工作包括：

- 讀取溫度、電壓與風扇轉速
- 根據溫度調整風扇
- 記錄硬體錯誤與系統事件
- 從遠端開機、關機或重新啟動伺服器
- 提供 IPMI 或 Redfish 管理介面
- 在 Host OS 無法使用時協助遠端除錯

### 一個簡化的系統圖

```text
Remote Administrator
        |
        | Management Network
        v
+-------------------+
|        BMC        |
| Sensors / Logs /  |
| Power Control     |
+-------------------+
        |
        | Hardware interfaces
        v
+-------------------+
| Server Hardware   |
| CPU / Fan / Power |
+-------------------+
        |
        v
+-------------------+
| Host OS           |
+-------------------+
```

BMC 可以觀察並控制伺服器硬體，但它不是 Host OS，也不是用來執行一般伺服器工作負載的主要 CPU。

---

## 3. Host OS 當機時，BMC 為什麼仍可工作？

因為 BMC 與 Host OS 使用不同的運算與執行環境。

Host OS 執行在伺服器的主要 CPU 與記憶體上；BMC 則使用自己的處理器、記憶體和 firmware。兩者可以互相溝通，但不依賴同一個作業系統才能執行。

此外，只要伺服器仍接著電源，BMC 通常就能從待機電源取得電力。即使 Host CPU 尚未開機，BMC 仍可能處於運作狀態。

因此，下列情況發生時，BMC 通常仍可工作：

- Host OS kernel panic
- Host OS 無法開機
- Host OS 正在重新啟動
- 主要 CPU 尚未啟動
- 伺服器處於關機但仍接電的狀態

### 要注意的限制

「獨立運作」不代表 BMC 永遠不會故障。若 BMC 自己當機、失去待機電源，或硬體介面故障，它也可能無法管理伺服器。

---

## 4. BMC 和一般 MCU firmware 有什麼不同？

MCU 是 **Microcontroller Unit**。一般 MCU firmware 常負責控制一個範圍較小、目的明確的硬體功能，例如：

- 讀取按鈕
- 控制 LED
- 驅動馬達
- 讀取單一感測器
- 執行即時控制邏輯

BMC 則負責整台伺服器的管理，是一個較完整的管理系統。它通常需要同時處理硬體監控、網路 API、權限、事件紀錄、服務恢復等工作。

| 比較項目 | 一般 MCU firmware | BMC firmware |
|---|---|---|
| 主要角色 | 控制特定硬體功能 | 管理整台伺服器 |
| 系統複雜度 | 通常較小 | 通常較高 |
| 執行環境 | Bare metal 或 RTOS 常見 | Embedded Linux 常見 |
| 網路管理 API | 不一定需要 | 通常需要 IPMI、Redfish 等介面 |
| 功能範圍 | 單一或少數功能 | Sensor、power、fan、logging、security 等 |
| 軟體架構 | Main loop、interrupt、task | 多個 service、IPC、API、daemon |

### 重要觀念

BMC 與 MCU firmware 並不是完全互斥的分類。BMC 描述的是「伺服器管理控制器的角色」，MCU 描述的是一種硬體類型。

早期或簡單的 BMC 可能使用較小型控制器；現代 BMC 通常使用能執行 Embedded Linux 的 SoC。面試時應避免只回答「BMC 跑 Linux，MCU 不跑 Linux」，因為這不是絕對規則。

---

## 5. IPMI 是什麼？

IPMI 是 **Intelligent Platform Management Interface**。

它是一套較早期、廣泛使用的伺服器管理規範。管理工具可以透過 IPMI 查詢硬體狀態、讀取事件紀錄，以及執行遠端電源控制。

常見用途：

- 查詢 sensor 數值
- 查詢 System Event Log
- 遠端開機、關機或重新啟動
- 取得伺服器管理資訊

你目前只需要記住：

> IPMI 是一套傳統且常見的伺服器硬體管理介面。

---

## 6. Redfish 是什麼？

Redfish 是較現代的伺服器管理標準。它使用常見的 Web 技術，例如：

- HTTP / HTTPS
- REST-style API
- JSON

簡化的 Redfish request 可能看起來像：

```http
GET /redfish/v1/Systems/1
```

回應可能是 JSON：

```json
{
  "Id": "1",
  "PowerState": "On",
  "Status": {
    "Health": "OK"
  }
}
```

你目前只需要記住：

> Redfish 使用 Web API 與 JSON，讓伺服器管理更容易與現代軟體整合。

---

## 7. IPMI 與 Redfish 的基礎比較

| 比較項目 | IPMI | Redfish |
|---|---|---|
| 定位 | 傳統伺服器管理介面 | 現代伺服器管理 API |
| 常見資料形式 | 二進位協定與命令 | JSON |
| 常見傳輸方式 | LAN、系統介面等 | HTTPS |
| 對 Web 開發者 | 較陌生 | 較容易理解與整合 |
| 使用狀況 | 舊系統與既有工具仍常見 | 現代管理平台常使用 |

兩者都可以用來管理伺服器。實際產品中可能同時支援 IPMI 與 Redfish，而不是只能選擇其中一個。

Mini-BMC 專案會先設計 REST API，因為它比較容易學習，也能幫助你理解 Redfish 類型的管理介面。

---

## 8. 將概念連接到 Mini-BMC 專案

Mini-BMC 不會真的控制伺服器硬體，而是模擬 BMC 的核心行為：

| 真實 BMC 功能 | Mini-BMC 對應功能 |
|---|---|
| 讀取實體溫度感測器 | Sensor simulator 產生溫度 |
| 控制實體風扇 | Fan service 計算風扇速度 |
| 控制伺服器電源 | Power state machine 模擬狀態轉換 |
| 提供 Redfish API | 提供簡化 REST API |
| 記錄硬體事件 | Structured logging 與 event system |
| 監控服務健康 | Watchdog 與 health check |

因此，這個專案的重點不是製造真正的 BMC，而是學習 BMC 軟體背後的架構、可靠性與管理流程。

---

## 9. Day 1 三題參考答案

請先嘗試自己回答，再閱讀以下內容。

### 問題一：BMC 是什麼？

BMC 是伺服器主機板上的獨立管理控制器。它負責監控溫度、風扇、電壓等硬體狀態，也能記錄事件及遠端控制伺服器電源。管理員可以透過 IPMI 或 Redfish 等介面操作它。

### 問題二：BMC 和一般 MCU firmware 有什麼不同？

一般 MCU firmware 通常負責範圍較小且明確的硬體控制工作；BMC 則負責整台伺服器的管理，需要處理 sensor、電源、風扇、網路 API、事件紀錄與權限等功能。現代 BMC 通常是一個執行 Embedded Linux、包含多個 service 的管理系統，但兩者的界線並非絕對。

### 問題三：Host OS 當機時，BMC 為什麼仍可工作？

BMC 有自己的處理器、記憶體、firmware 與電源來源，不依賴 Host OS 執行。即使 Host OS 或主要 CPU 無法工作，只要 BMC 本身與待機電源正常，它仍可監控硬體並接受遠端管理命令。

---

## 10. 自我測驗

先遮住答案，嘗試口頭回答。

1. BMC 的主要工作是什麼？
2. BMC 和 Host OS 執行在同一個處理器上嗎？
3. 伺服器關機後，BMC 是否一定也會關機？
4. 為什麼遠端電源控制對資料中心很重要？
5. IPMI 和 Redfish 都能解決什麼問題？
6. Redfish 為什麼容易與現代管理軟體整合？
7. 為什麼不能簡單地說「BMC 一定跑 Linux，MCU 一定不跑 Linux」？
8. Mini-BMC 專案和真實 BMC 有什麼差異？

### 簡短答案

1. 獨立監控與管理伺服器硬體。
2. 通常不是，BMC 有自己的處理器或 SoC。
3. 不一定；只要仍接著電源，BMC 通常可由待機電源繼續運作。
4. 管理員不必到現場，就能處理無法開機或當機的伺服器。
5. 查詢伺服器狀態並執行管理操作。
6. 它使用 HTTPS、REST-style API 與 JSON。
7. 因為 BMC 是系統角色，MCU 是硬體類型，而且實際設計有很多變化。
8. Mini-BMC 使用軟體模擬硬體與管理流程，不直接控制真實伺服器。

---

## 11. 完成 Day 1 的方式

不要背誦參考答案。請建立 `docs/learning_notes/week1.md`，用自己的話寫下：

```markdown
# Week 1 學習筆記

## BMC 是什麼？

## BMC 和一般 MCU firmware 有什麼不同？

## Host OS 當機時，BMC 為什麼仍可工作？

## IPMI 與 Redfish 的差異

## 我還不理解的問題
```

每題寫 3-5 句即可。寫完後，不看文件口頭回答前三題；能清楚回答，就可以勾選 Day 1 對應任務。
