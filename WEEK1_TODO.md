# Mini-BMC 第一週 TODO

> 第一週目標：理解 BMC / OpenBMC / Linux service 基礎，並完成 Mini-BMC 的文件與可編譯 C++ 骨架。
>
> 建議投入時間：12-18 小時。完成任務後，將 `[ ]` 改成 `[x]`，並在「學習筆記」留下你自己的理解。

## 本週完成標準

- [ ] 能用自己的話，在 2 分鐘內解釋 BMC、OpenBMC、IPMI 與 Redfish
- [ ] 能解釋 process、thread、daemon、service 與 systemd 的關係
- [ ] 完成 `README.md`
- [ ] 完成 `docs/architecture.md`
- [ ] 完成 `docs/api_spec.md`
- [ ] C++ 專案可以使用 CMake configure、build 並執行

## Day 1：確認方向與 BMC 基礎（2-3 小時）

- [x] 建立專案 repo
- [x] 建立完整專案與學習計畫：`BMC_Project_and_Learning_Plan.md`
- [ ] 閱讀 Day 1 學習文件：`docs/learning/day1_bmc_basics.md`
- [ ] 閱讀 BMC overview，理解 BMC 為何能獨立於 host OS 運作
- [ ] 閱讀 IPMI 與 Redfish overview，整理兩者差異
- [ ] 用自己的話回答：
  - BMC 是什麼？
  - BMC 和一般 MCU firmware 有什麼不同？
  - Host OS 當機時，BMC 為什麼仍可工作？
- [ ] 將答案寫入 `docs/learning_notes/week1.md`

完成條件：不看資料也能口頭解釋上述三題。

## Day 2：OpenBMC 與 Linux Service 基礎（2-3 小時）

- [ ] 閱讀 OpenBMC architecture overview
- [ ] 找出 OpenBMC 中 bmcweb、D-Bus、sensor service 的基本角色
- [ ] 學習 process、thread、daemon、service 的差異
- [ ] 學習 systemd 基本操作：
  - `systemctl status <service>`
  - `systemctl start|stop|restart <service>`
  - `journalctl -u <service>`
- [ ] 將理解與常用指令寫入 `docs/learning_notes/week1.md`

完成條件：能畫出簡化的 OpenBMC request flow，並解釋 systemd 如何管理 service。

## Day 3：定義專案範圍與 README（2 小時）

- [ ] 建立 `README.md`
- [ ] README 說明專案目的、核心功能、技術選型與 12 週 roadmap
- [ ] 明確定義第一版只支援單一 server node
- [ ] 明確定義第一版使用 C++20、CMake，REST library 暫緩至 Week 7 決定
- [ ] 加入預計的 build 與 run 指令

完成條件：第一次看到 repo 的人，能在 5 分鐘內理解專案要解決什麼問題。

## Day 4：系統架構設計（2-3 小時）

- [ ] 建立 `docs/architecture.md`
- [ ] 使用 Mermaid 畫出 Mini-BMC 高階架構圖
- [ ] 說明以下模組責任：
  - Sensor Service
  - Event Bus
  - Fan Control Service
  - Power Service
  - Watchdog
  - Management API
  - Logging / Metrics
- [ ] 說明 service 間使用 event bus 的原因
- [ ] 記錄第一版採用單一 process、多個模組的決策與 trade-off

完成條件：能說明每個模組負責什麼，以及事件如何在系統中流動。

## Day 5：API 規格初版（2 小時）

- [ ] 建立 `docs/api_spec.md`
- [ ] 定義共同 JSON response 與 error response 格式
- [ ] 定義以下 API 的 request、response、HTTP status：
  - `GET /health`
  - `GET /status`
  - `GET /sensors`
  - `GET /power/status`
  - `POST /power/on`
  - `POST /power/off`
  - `POST /power/reboot`
  - `GET /metrics`
- [ ] 記錄 illegal power transition 預計如何回應

完成條件：每個 endpoint 都有至少一組成功與失敗 response 範例。

## Day 6：建立 C++ / CMake 骨架（2-3 小時）

- [ ] 建立根目錄 `CMakeLists.txt`
- [ ] 建立 `src/main.cpp`
- [ ] 建立預計使用的模組目錄
- [ ] 建立 `tests/` 目錄
- [ ] 程式啟動時輸出 Mini-BMC 啟動訊息後正常結束
- [ ] 執行並通過：

```bash
cmake -S . -B build
cmake --build build
./build/mini-bmc
```

完成條件：從乾淨的 `build/` 目錄可成功 configure、build、run。

## Day 7：複習與本週驗收（1-2 小時）

- [ ] 逐項檢查「本週完成標準」
- [ ] 修正文件中不一致的命名與設計
- [ ] 將本週遇到的問題與答案補進 `docs/learning_notes/week1.md`
- [ ] 寫下三個仍不理解、下週需要釐清的問題
- [ ] 建立 Week 1 完成 commit

完成條件：文件和骨架都在 repo 中，且你能不看稿解釋本週的核心概念。

## 建議初始目錄

```text
.
├── CMakeLists.txt
├── README.md
├── BMC_Project_and_Learning_Plan.md
├── WEEK1_TODO.md
├── docs/
│   ├── api_spec.md
│   ├── architecture.md
│   └── learning_notes/
│       └── week1.md
├── src/
│   └── main.cpp
└── tests/
```

## 學習提醒

- 每個主題先自己寫 3-5 句理解，再用資料或 AI 修正。
- 第一週先把架構與名詞理解清楚，不急著實作 sensor、event bus 或 REST server。
- 每完成一天，立即勾選任務；只有達成「完成條件」才算完成該日。
