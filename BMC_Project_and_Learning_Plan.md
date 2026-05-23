# BMC / Embedded Linux 專案與學習計畫

## 0. 目標定位

這份計畫的目標不是做一個普通的 side project，而是建立一個可以拿去面試 BMC Firmware / Embedded Linux / Server Platform Engineer 的系統型專案。

核心策略分成兩階段：

1. **從零建立 Mini-BMC / Server Management Platform**
   - 建立 BMC 架構概念
   - 熟悉 Linux service、sensor monitoring、fan control、power control、logging、fault handling
   - 做出可以展示、可以講架構、可以分析 trade-off 的作品

2. **閱讀並修改 OpenBMC 相關 codebase**
   - 理解真實 production-grade BMC firmware stack
   - 研究 OpenBMC 如何處理 Redfish API、sensor、fan control、logging、D-Bus service
   - 嘗試修改小功能或建立對照版 implementation

最終成果應該能讓面試官相信：

> 你不是只會寫 demo，而是理解 Linux-based firmware/service architecture、debugging、reliability、observability，以及大型 codebase 的基本工程流程。

---

## 1. BMC 是什麼：專案背景理解

BMC（Baseboard Management Controller）可以理解成伺服器主機板上的一台小型管理電腦。即使主機 OS 當機，BMC 仍然可以獨立運作，負責：

- 遠端開關機
- Sensor monitoring：CPU 溫度、風扇轉速、電壓、電源狀態
- Fan speed control
- Power control
- Firmware update
- 系統事件紀錄
- Redfish / IPMI API
- 遠端管理與除錯

BMC 工程師常碰的不是單純 MCU register，而是：

- Linux / Embedded Linux
- systemd service
- D-Bus
- REST API / Redfish
- C / C++ service
- sensor abstraction
- fault handling
- kernel config / device tree / driver porting
- debugging and logging

---

# Part A：自建 Mini-BMC Project

## 2. 專案名稱

**OpenBMC-inspired Mini Server Management Platform**

簡稱：**Mini-BMC**

---

## 3. 專案應該做出什麼東西

這個專案應該是一個簡化版 BMC 系統，能模擬一台或多台 server node 的硬體狀態，並提供管理、監控、控制、故障恢復與 log/metrics 功能。

### 3.1 系統架構

```text
                    Web Dashboard / CLI Client
                              |
                       REST / WebSocket
                              |
                    Management API Service
                              |
         ------------------------------------------------
         |                    |                         |
  Sensor Service        Fan Control Service       Power Service
         |                    |                         |
         ---------------- Event / Message Bus ----------
                              |
                    Logging / Metrics / Watchdog
                              |
                    Sensor Simulator / Node Simulator
```

### 3.2 必做功能

#### 1. Sensor Monitoring

模擬以下 sensor：

- CPU temperature
- Fan speed
- Power usage
- Voltage
- Node health status

功能要求：

- 定期 polling sensor
- sensor threshold policy
- warning / critical 狀態判斷
- sensor timeout 模擬
- retry mechanism
- abnormal sensor value detection

範例：

```text
CPU temp < 70°C      -> OK
70°C <= temp < 85°C  -> WARNING
temp >= 85°C         -> CRITICAL
```

---

#### 2. Fan Control

根據溫度調整 fan speed。

範例 policy：

```text
temp < 60°C      -> fan 30%
60°C - 75°C      -> fan 60%
75°C - 85°C      -> fan 80%
temp >= 85°C     -> fan 100%
```

進階要求：

- hysteresis，避免風扇轉速因溫度邊界頻繁震盪
- fan failure detection
- manual override mode
- auto mode / manual mode 切換

---

#### 3. Power Control

建立 server power state machine。

狀態：

```text
OFF
BOOTING
ON
SHUTTING_DOWN
REBOOTING
ERROR
```

支援 API：

```http
POST /power/on
POST /power/off
POST /power/reboot
GET  /power/status
```

重點不是單純改變變數，而是要做出：

- 狀態轉移限制
- illegal transition handling
- timeout handling
- error state recovery

例如：

```text
OFF -> BOOTING -> ON
ON -> SHUTTING_DOWN -> OFF
ON -> REBOOTING -> BOOTING -> ON
```

---

#### 4. Event System

不要所有模組直接互相呼叫，應該建立 event queue / event dispatcher。

事件範例：

```text
TEMP_WARNING
TEMP_CRITICAL
FAN_FAILURE
SENSOR_TIMEOUT
NODE_HEARTBEAT_LOST
POWER_STATE_CHANGED
SERVICE_RESTARTED
```

學習重點：

- event-driven architecture
- producer-consumer pattern
- queue
- concurrency
- fault isolation
- decoupling

---

#### 5. Watchdog / Health Check

建立 service heartbeat 機制。

功能：

- 每個 service 定期送 heartbeat
- 若超過 N 秒未回應，標記為 degraded
- 嘗試 restart service
- 紀錄 recovery log
- 更新 node health status

範例：

```text
sensor-service heartbeat lost > 5 sec
-> mark service DEGRADED
-> restart sensor-service
-> log SERVICE_RESTARTED
```

這是整個專案最接近 BMC/infra reliability 的部分之一。

---

#### 6. Logging System

不要只用 print。應該使用 structured logging。

Log 格式建議：

```json
{
  "timestamp": "2026-05-22T12:00:00Z",
  "service": "sensor-service",
  "level": "ERROR",
  "event": "SENSOR_TIMEOUT",
  "message": "CPU temperature sensor timeout after 3 retries"
}
```

功能：

- log level：DEBUG / INFO / WARNING / ERROR / CRITICAL
- log file output
- log rotation
- query recent logs
- API 查詢 logs

API 範例：

```http
GET /logs?level=ERROR
GET /logs?service=sensor-service
```

---

#### 7. Metrics / Observability

輸出 Prometheus-style metrics。

範例：

```text
mini_bmc_cpu_temperature_celsius 78
mini_bmc_fan_speed_percent 80
mini_bmc_event_queue_size 12
mini_bmc_sensor_timeout_total 3
mini_bmc_service_restart_total 1
mini_bmc_heartbeat_latency_ms 42
```

功能：

```http
GET /metrics
GET /health
GET /status
```

學習重點：

- observability
- system health
- latency
- failure count
- recovery count
- monitoring mindset

---

#### 8. Fault Injection

這是讓專案有深度的關鍵。

支援主動注入錯誤：

```http
POST /fault/sensor-timeout
POST /fault/high-temperature
POST /fault/fan-failure
POST /fault/service-crash
POST /fault/heartbeat-lost
POST /fault/network-delay
```

每個 fault injection 都應該能觸發：

- event
- log
- metric
- health status change
- recovery action

這會讓專案從普通 monitoring app 變成 reliability/system engineering project。

---

#### 9. Configuration System

使用 YAML / JSON 作為設定檔。

範例：

```yaml
sensor:
  polling_interval_ms: 1000
  cpu_temp_warning: 70
  cpu_temp_critical: 85
  retry_count: 3

fan:
  hysteresis: 5
  default_mode: auto

watchdog:
  heartbeat_timeout_ms: 5000
  restart_limit: 3
```

進階：

- runtime reload config
- config validation
- invalid config handling

---

#### 10. Deployment

專案不應該只是在 local terminal 跑起來，而是要像真實 service。

至少要有：

- CMake build
- Dockerfile
- Docker Compose
- systemd service file
- README deployment guide

建議支援兩種模式：

1. **Local development mode**
2. **Linux service mode**

---

## 4. 技術選型建議

### 4.1 推薦主體語言：C++

理由：

- BMC / firmware / embedded Linux 常見
- 可以練 C++ service architecture
- 可以練 threading、socket、memory、RAII
- 比純 Python 更有面試價值

### 4.2 可用技術

| 模組 | 建議技術 |
|---|---|
| Core service | C++17 / C++20 |
| Build system | CMake |
| REST API | Crow / Pistache / cpp-httplib |
| Logging | spdlog |
| Config | yaml-cpp / nlohmann-json |
| Metrics | 自行實作 Prometheus text format |
| Test | GoogleTest |
| Container | Docker / Docker Compose |
| Linux service | systemd |
| Dashboard | 可選，React / simple HTML |
| Simulator | C++ 或 Python |

### 4.3 是否要做 Dashboard？

可以做，但不是重點。

Dashboard 只需要簡單呈現：

- sensor value
- power state
- fan speed
- service health
- recent events
- fault injection buttons

不要把時間花在 UI 美化。重點是 backend architecture。

---

## 5. 專案學習點

### 5.1 系統設計

你應該能回答：

- 為什麼要拆成多個 service？
- polling 和 event-driven 差在哪？
- state machine 怎麼設計？
- fault 怎麼 propagate？
- service crash 後系統怎麼恢復？
- log 和 metrics 各自解決什麼問題？

### 5.2 Linux / Embedded Linux

你會學到：

- process / thread
- systemd service
- journalctl
- daemon process
- signal handling
- file permissions
- config file loading
- service restart

### 5.3 C++ / System Programming

你會學到：

- class design
- RAII
- threading
- mutex / condition variable
- queue
- socket / HTTP server
- exception handling
- memory ownership
- testing

### 5.4 Reliability / Observability

你會學到：

- watchdog
- heartbeat
- retry
- timeout
- degradation
- auto recovery
- metrics
- structured logging
- fault injection

### 5.5 BMC Domain Knowledge

你會學到：

- sensor monitoring
- fan control
- power state
- Redfish-like API
- hardware abstraction
- service-based management architecture

---

# Part B：大型 Codebase 閱讀與修改

## 6. 建議使用的 codebase

最推薦使用 OpenBMC 相關 codebase，而不是隨便找 toy repo。

### 6.1 第一優先：OpenBMC / bmcweb

Repo：

```text
https://github.com/openbmc/bmcweb
```

用途：

- OpenBMC 的 Redfish / REST API server
- C++
- async HTTP
- JSON response
- BMC 管理 API
- 面試價值高

適合原因：

- 和 BMC JD 的 REST API / Redfish 直接相關
- CS 背景較容易切入
- 不需要一開始碰 Yocto build 全家桶
- 可以對照你 Mini-BMC 的 API design

建議修改方向：

1. 新增一個 mock health endpoint
2. 新增 debug-only fault injection endpoint
3. 新增 sensor summary endpoint
4. 加入 API latency logging
5. trace 某個 Redfish endpoint 的 request flow

學習點：

- Redfish API 架構
- HTTP server in embedded environment
- JSON schema
- async request handling
- OpenBMC service 如何透過 D-Bus 取得資料
- production API code style

---

### 6.2 第二優先：OpenBMC / dbus-sensors

Repo：

```text
https://github.com/openbmc/dbus-sensors
```

用途：

- OpenBMC sensor service
- 讀取 hwmon / sensor data
- 將 sensor 暴露到 D-Bus

適合原因：

- 和 Sensor Monitoring 直接相關
- 可以理解 OpenBMC sensor abstraction
- 可對照你自己的 Sensor Service

建議修改方向：

1. 加入 mock sensor backend
2. 新增 sensor timeout log
3. 新增 sensor threshold policy
4. 加入簡單 metrics counter
5. trace sensor value 從讀取到 D-Bus exposure 的流程

學習點：

- sensor abstraction
- Linux hwmon
- D-Bus object model
- async sensor polling
- hardware data 如何被 service 化

---

### 6.3 第三優先：OpenBMC / phosphor-pid-control

Repo：

```text
https://github.com/openbmc/phosphor-pid-control
```

用途：

- Fan control / thermal control
- PID control loop
- 根據 sensor 控制 fan

適合原因：

- 和 Fan Speed Control 直接相關
- 有控制邏輯與 policy 設計
- 可以對照你自己做的 fan control

建議修改方向：

1. 新增簡化 fan policy
2. 加入 hysteresis 設計
3. 加入 abnormal sensor fallback behavior
4. trace control loop flow
5. 寫一份 fan control policy analysis

學習點：

- control loop
- thermal management
- sensor-to-actuator control
- fallback policy
- stability / oscillation 問題

---

### 6.4 第四優先：OpenBMC / phosphor-logging

Repo：

```text
https://github.com/openbmc/phosphor-logging
```

用途：

- OpenBMC logging / error handling

適合原因：

- 對應你的 structured logging 與 fault handling
- 可以學 production logging 設計

建議修改方向：

1. trace error event 如何被記錄
2. 新增一類 mock error
3. 對照 Mini-BMC 的 logging design
4. 寫一份 OpenBMC logging flow note

學習點：

- error model
- event logging
- persistent log
- production log handling

---

## 7. 大型 codebase 階段的正確目標

不要一開始就追求 upstream PR merged。

更合理的目標是：

1. 能 build 或至少理解模組 build flow
2. 能 trace 一條 request / event flow
3. 能畫出架構圖
4. 能修改一個小功能
5. 能寫出 technical note
6. 能把 OpenBMC 設計和你的 Mini-BMC 對照

面試時最有價值的不是：

> 我改了幾行 code。

而是：

> 我理解 production BMC stack 的 service decomposition、D-Bus communication、Redfish API flow，並在自己的 Mini-BMC 中重現部分概念，進一步比較 polling/event-driven、logging/metrics、watchdog/recovery 的設計差異。

---

# Part C：完整學習時程規劃

## 8. 預計投入時間

建議投入時間：**12 週，每週 12–18 小時**

總投入：約 **150–220 小時**

如果時間更充裕，可以拉長到 16 週，補 OpenBMC / Yocto / D-Bus。

---

## 9. 12 週計畫

### Week 1：BMC / Linux / OpenBMC 概念建立

目標：

- 理解 BMC 是什麼
- 理解 OpenBMC 在做什麼
- 熟悉 Linux service 基礎

應讀內容：

- BMC / IPMI / Redfish overview
- OpenBMC architecture overview
- Linux systemd basics
- process / service / daemon 概念

實作：

- 建立 repo
- 寫 project README 初版
- 畫系統架構圖
- 決定 API spec
- 建立 CMake skeleton

產出：

- `README.md`
- `docs/architecture.md`
- `docs/api_spec.md`
- empty C++ project skeleton

---

### Week 2：C++ 基礎服務框架

目標：

- 建立 core service structure
- 建立 config/logging 基礎

實作：

- CMake
- config loader
- structured logger
- base service class
- service lifecycle：start / stop / health

產出：

- `src/core/`
- `src/config/`
- `src/logging/`
- 基本單元測試

必須自己理解：

- CMake
- class design
- RAII
- logging design
- config validation

AI 可協助：

- CMake template
- logger wrapper boilerplate
- README 格式

---

### Week 3：Sensor Service

目標：

- 實作 sensor simulation 與 polling

實作：

- CPU temperature sensor
- fan speed sensor
- power usage sensor
- sensor polling
- threshold status
- retry / timeout

產出：

- `sensor-service`
- `docs/sensor_design.md`
- sensor unit tests

必須自己理解：

- polling loop
- timeout
- retry
- threshold policy
- abnormal data handling

AI 可協助：

- test cases 生成
- sensor fake data generator
- code cleanup

---

### Week 4：Event System

目標：

- 建立 event queue / dispatcher

實作：

- event type
- event queue
- producer-consumer model
- event handler registration
- event log integration

產出：

- `event-bus`
- `docs/event_architecture.md`
- event flow diagram

必須自己理解：

- queue
- mutex
- condition variable
- producer-consumer
- why event-driven

AI 可協助：

- 畫圖用 Mermaid
- 產生事件 enum 初稿
- 補文件

---

### Week 5：Fan Control

目標：

- 實作 fan control service

實作：

- temperature-to-fan policy
- hysteresis
- fan failure detection
- auto/manual mode
- event trigger

產出：

- `fan-control-service`
- `docs/fan_control.md`
- policy tests

必須自己理解：

- hysteresis 為什麼需要
- control policy
- sensor-to-actuator relation
- fallback behavior

AI 可協助：

- 產生 policy table
- 產生測試案例
- 協助 refactor

---

### Week 6：Power State Machine

目標：

- 實作 power control service

實作：

- state machine
- legal transition
- illegal transition handling
- reboot flow
- timeout handling

產出：

- `power-service`
- `docs/power_state_machine.md`
- state machine diagram

必須自己理解：

- finite state machine
- state transition correctness
- race condition
- timeout/error state

AI 可協助：

- Mermaid state diagram
- API document
- exception message 改寫

---

### Week 7：REST API / Redfish-like API

目標：

- 建立管理 API

實作：

```http
GET /health
GET /sensors
GET /sensors/{id}
GET /power/status
POST /power/on
POST /power/off
POST /power/reboot
GET /events
GET /logs
GET /metrics
POST /fault/{type}
```

產出：

- `management-api`
- `docs/api_spec.md`
- curl examples

必須自己理解：

- HTTP
- REST
- JSON schema
- API error handling
- Redfish 的基本概念

AI 可協助：

- 產生 OpenAPI spec 初稿
- 產生 curl examples
- 產生 API response 格式

---

### Week 8：Watchdog / Fault Injection

目標：

- 專案深度核心

實作：

- service heartbeat
- watchdog monitor
- service degraded status
- restart simulation
- fault injection API
- recovery log

產出：

- `watchdog-service`
- `fault-injection`
- `docs/fault_handling.md`

必須自己理解：

- heartbeat
- watchdog
- degradation
- recovery policy
- failure propagation
- false positive / detection latency

AI 可協助：

- 產生 fault scenario 清單
- 產生 log message template
- 補測試

---

### Week 9：Metrics / Observability / Deployment

目標：

- 讓專案像 production service

實作：

- `/metrics`
- latency metrics
- event count
- restart count
- Dockerfile
- Docker Compose
- systemd service file
- journalctl demo

產出：

- `docs/deployment.md`
- `docker-compose.yml`
- `mini-bmc.service`
- metrics examples

必須自己理解：

- metrics 的意義
- observability vs logging
- systemd service lifecycle
- Docker 只是部署，不是核心

AI 可協助：

- Dockerfile
- docker-compose
- systemd unit file 初稿
- README deployment guide

---

### Week 10：Benchmark / Analysis

目標：

- 讓專案有碩論味

實驗設計：

1. Polling interval vs CPU usage / detection latency
2. Event queue size under fault burst
3. Watchdog timeout vs recovery latency
4. Sensor retry count vs false alarm
5. Fan hysteresis vs oscillation frequency

產出：

- `docs/experiments.md`
- `results/`
- latency / CPU usage / event count 表格
- 分析圖

必須自己理解：

- 實驗假設
- 指標定義
- trade-off
- 結果解讀

AI 可協助：

- 實驗紀錄模板
- Python 畫圖 code
- 表格整理
- 英文報告潤稿

---

### Week 11：OpenBMC Codebase Study

目標：

- 進入真實 codebase

建議順序：

1. bmcweb
2. dbus-sensors
3. phosphor-pid-control
4. phosphor-logging

任務：

- clone repo
- 看 README / docs
- 找一條 request/event flow
- 做 architecture note
- 和 Mini-BMC 對照

產出：

- `docs/openbmc_bmcweb_study.md`
- `docs/openbmc_sensor_study.md`
- `docs/openbmc_comparison.md`

必須自己理解：

- OpenBMC 為什麼用 D-Bus
- Redfish endpoint flow
- sensor 如何被 abstract
- fan control 和你的簡化版差在哪

AI 可協助：

- 解釋陌生 C++ code
- 幫你整理 call graph
- 幫你把理解轉成文件
- 但不能代替你 trace code

---

### Week 12：OpenBMC 小修改 + Final Report

目標：

- 做一個可展示的 codebase modification

可選修改：

#### Option A：bmcweb

新增 debug-only endpoint：

```http
GET /redfish/v1/Managers/bmc/HealthSummary
```

或加 latency logging。

#### Option B：dbus-sensors

新增 mock sensor backend 或 timeout log。

#### Option C：phosphor-pid-control

新增簡化 hysteresis policy 或 fallback behavior。

產出：

- forked repo
- branch
- commit history
- modification note
- final technical report

Final report 架構：

```text
1. Introduction
2. Background: BMC and OpenBMC
3. Mini-BMC System Architecture
4. Implementation
5. Fault Handling and Observability
6. Evaluation
7. OpenBMC Codebase Study
8. Comparison with OpenBMC
9. Lessons Learned
10. Future Work
```

---

# Part D：哪些地方要自己讀懂，哪些可以交給 AI

## 10. 必須自己完全理解的部分

這些是面試官很容易追問的，不能只靠 AI。

### 10.1 BMC / OpenBMC 基本概念

你要能自己講清楚：

- BMC 是什麼
- 為什麼 BMC 可以在 host OS 掛掉時仍然工作
- IPMI / Redfish 是什麼
- OpenBMC 和 AMI BMC 大概差在哪
- BMC 和一般 MCU firmware 差在哪

---

### 10.2 Linux service / process

你要理解：

- process vs thread
- daemon 是什麼
- systemd 怎麼管理 service
- journalctl 怎麼看 log
- service crash 怎麼 debug
- signal handling

---

### 10.3 C++ core logic

你要自己理解：

- thread / mutex / condition variable
- queue
- state machine
- memory ownership
- class design
- error handling

不能完全讓 AI 寫完後不懂。

---

### 10.4 System architecture

你要能自己解釋：

- 為什麼要 event bus
- 為什麼要 watchdog
- 為什麼要 metrics
- 為什麼 logging 不等於 observability
- fault injection 的意義
- polling vs event-driven trade-off

---

### 10.5 OpenBMC code flow

你不需要懂整個 OpenBMC，但你至少要能講：

- bmcweb 某個 endpoint 怎麼處理 request
- dbus-sensors 如何讀 sensor 並暴露到 D-Bus
- phosphor-pid-control 如何根據 sensor 控制 fan
- 你的 Mini-BMC 對應 OpenBMC 哪些概念

---

## 11. 可以讓 AI 協助的部分

AI 可以用來加速，但不能代替理解。

### 11.1 Boilerplate code

例如：

- CMakeLists.txt
- Dockerfile
- systemd service file
- basic REST API skeleton
- config parser wrapper
- logging wrapper

### 11.2 測試資料與測試案例

例如：

- 產生 sensor simulation data
- 產生 unit test cases
- 產生 fault scenario
- 產生 curl examples

### 11.3 文件整理

例如：

- README 潤稿
- API spec 初稿
- architecture diagram Mermaid
- final report 英文潤稿
- 面試用 project summary

### 11.4 Code explanation

你可以把陌生 code 丟給 AI 問：

- 這段在幹嘛？
- call flow 是什麼？
- 這個 class 負責什麼？
- 這裡可能有哪些 race condition？
- 這個 design 和我的 Mini-BMC 差在哪？

但最後一定要自己 trace 一次。

---

## 12. 不建議完全交給 AI 的部分

以下如果交給 AI 代寫，專案會變得很空：

- event system architecture
- state machine 設計
- watchdog recovery policy
- benchmark 設計
- OpenBMC code flow 理解
- final report 的 technical argument
- 面試時要講的 trade-off

這些是專案含金量所在。

---

# Part E：面試展示方式

## 13. GitHub Repo 應該長怎樣

建議結構：

```text
mini-bmc/
├── README.md
├── CMakeLists.txt
├── docker-compose.yml
├── systemd/
│   └── mini-bmc.service
├── config/
│   └── config.yaml
├── src/
│   ├── core/
│   ├── sensor/
│   ├── fan/
│   ├── power/
│   ├── event/
│   ├── watchdog/
│   ├── api/
│   ├── logging/
│   └── metrics/
├── tests/
├── docs/
│   ├── architecture.md
│   ├── api_spec.md
│   ├── sensor_design.md
│   ├── fan_control.md
│   ├── power_state_machine.md
│   ├── fault_handling.md
│   ├── experiments.md
│   ├── openbmc_bmcweb_study.md
│   ├── openbmc_sensor_study.md
│   └── openbmc_comparison.md
└── results/
```

---

## 14. README 必須包含

README 不要只是安裝教學，應該像 engineering project。

內容：

1. Project motivation
2. BMC background
3. System architecture diagram
4. Features
5. API examples
6. Fault injection demo
7. Metrics example
8. Deployment guide
9. Experiment results
10. OpenBMC comparison
11. Future work

---

## 15. 面試時 2 分鐘講法

可以這樣講：

> I built a simplified OpenBMC-inspired server management platform to understand how BMC firmware manages sensors, power states, fan control, logging, and recovery. The system includes a sensor monitoring service, fan control policy, power state machine, event bus, watchdog, structured logging, Prometheus-style metrics, and fault injection APIs.  
>
> After building the system from scratch, I studied OpenBMC components such as bmcweb, dbus-sensors, and phosphor-pid-control, and compared my simplified design with OpenBMC’s D-Bus-based service architecture.  
>
> The goal was not just to build a dashboard, but to understand Linux-based firmware architecture, service reliability, observability, and fault handling.

---

## 16. 面試官可能追問

你要準備回答：

1. 為什麼 BMC 適合用 Linux-based architecture？
2. Redfish 和 REST API 的關係？
3. 你的 sensor service 如何處理 timeout？
4. fan control 為什麼需要 hysteresis？
5. power state machine 如何避免非法狀態？
6. watchdog 怎麼避免 false positive？
7. logging 和 metrics 有什麼差異？
8. polling 和 event-driven 哪個比較好？
9. OpenBMC 為什麼使用 D-Bus？
10. 你修改 OpenBMC codebase 時遇到什麼困難？
11. 你的 Mini-BMC 和 OpenBMC 最大差異是什麼？
12. 如果要接真實硬體，你會怎麼改？

---

# Part F：加分延伸

## 17. 進階方向

如果 12 週完成後還有時間，可以加：

### 17.1 D-Bus Integration

把 Mini-BMC 的 service communication 改成 D-Bus。

學習價值很高，因為 OpenBMC 大量使用 D-Bus。

---

### 17.2 Linux hwmon

讀取真實 Linux sensor path：

```text
/sys/class/hwmon/
```

這會讓 sensor service 更接近真實 embedded Linux。

---

### 17.3 QEMU / ASPEED Emulation

嘗試用 QEMU 跑 OpenBMC image。

這很硬，但面試價值很高。

---

### 17.4 CI/CD

加入：

- GitHub Actions
- unit test
- clang-format
- clang-tidy
- cppcheck

這可以展示工程品質。

---

### 17.5 eBPF / tracing

進階 observability，可以研究 service latency / syscall tracing。

---

# 18. 最終成果檢查表

完成後你應該要有：

- [ ] Mini-BMC 可執行系統
- [ ] Sensor monitoring
- [ ] Fan control
- [ ] Power state machine
- [ ] REST API
- [ ] Event bus
- [ ] Watchdog
- [ ] Fault injection
- [ ] Structured logging
- [ ] Metrics endpoint
- [ ] Docker / systemd deployment
- [ ] Unit tests
- [ ] Benchmark / experiment results
- [ ] OpenBMC codebase study notes
- [ ] 至少一個 OpenBMC repo 小修改
- [ ] Final technical report
- [ ] 面試用 2 分鐘 project pitch

---

# 19. 最後建議

這個 project 的重點不是功能堆疊，而是：

1. 能不能說清楚架構
2. 能不能解釋 fault handling
3. 能不能 trace service flow
4. 能不能對照 OpenBMC
5. 能不能講出 trade-off

只要做到這些，就已經比普通履歷上的 side project 強很多。

對 BMC / Embedded Linux / Server Platform 面試而言，這會是一個非常有說服力的作品。
