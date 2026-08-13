# `5G_NWDAF_Infrastructure` 建置與遷移計畫

建立日期：2026-08-05
最近更新：2026-08-13

狀態：三台 VM、Host Docker ML services、GPU runtime、雙 Path 設定、公開 lifecycle commands、
fresh-user full-core FL closed loop 與 log-correlated teardown 均已完成 runtime 驗證。目前下一個
規劃範圍是修正 logs／status 的 observability contract；Infrastructure repository 仍未設定 remote。

## 文件導覽

本文件保留目標、現況、原則與 repository 邊界，作為整體計畫入口。詳細內容依責任拆分如下：

- [Repository、component scope、submodule、topology 與資源](5g-nwdaf-infrastructure-plan/repository-and-topology.md)
- [設定檔、testbed definition 與 scenario](5g-nwdaf-infrastructure-plan/configuration.md)
- [Host／Guest scripts、build 與 runtime lifecycle](5g-nwdaf-infrastructure-plan/operations.md)
- [實作階段、驗證矩陣、變更範圍與決策](5g-nwdaf-infrastructure-plan/roadmap.md)
- [實作紀錄：基礎建置與 E2E](5g-nwdaf-infrastructure-plan/implementation-history-foundation.md)
- [實作紀錄：contract hardening 與文件整理](5g-nwdaf-infrastructure-plan/implementation-history-hardening.md)
- [Fresh acceptance、cleanup 與 observability 計畫](5g-nwdaf-infrastructure-plan/acceptance-and-observability.md)

## 1. 目的

本計畫定義一個新的、可公開釋出且可重現的 5G NWDAF 實驗環境。暫定 repository
名稱為 `5G_NWDAF_Infrastructure`。它將取代舊 `5G_Infrastructure` 作為新版
NWDAF full-core 實驗的整合層，但不直接繼承舊 repository history 或 working tree。

第一個完整支援目標是目前已在單機 runner 驗證的 three-NWDAF、two-TAI、two-UPF
federated-learning closed loop：

- NWDAF-A／B 分別服務 TAI A／B，負責 analytics、model consumption、accuracy
  reporting 與 FL Client；
- NWDAF-C 擁有模型，負責 Model Provision、Model Monitor coordination 與 FL Server；
- UE 經真實 UERANSIM registration、authentication、PDU Session、serving-SMF
  resolution 與 UPF selection；
- UPF Event Exposure、ADRF、model monitoring、federated retraining、model publication
  與 reprovision 形成可驗證的 E2E evidence chain。

本計畫同時處理：

- 新 integration repository 的責任與目錄；
- component submodule 與版本固定策略；
- 三台精簡 VM 與 Host ML containers 的角色、資源及網路邊界；
- component config、testbed topology 與本機 override 邊界；
- host orchestration 與 guest setup／build／service lifecycle；
- 從舊 testbed 遷移並最終退場的安全條件。

## 2. 現況與問題

舊 `5G_Infrastructure` 適合保存歷史實驗與實驗室網路設定，但不適合作為新版公開
環境的起點：

- 它不是目前使用者主要維護的 repository，且長期保存大量 tracked／untracked
  本地差異；
- source 有時由 guest provisioning 直接 clone，有時由 host submodule mount 後再
  copy，實際版本不易由 parent repository 完整回答；
- 多台 VM 各自保存 free5GC checkout、build cache 與可修改 source，造成重複占用和
  version drift；
- Vagrant provisioning、`setup.sh`、guest source tree 與 runtime state 共同決定
  最終狀態，重建邊界不清楚；
- 舊雙 path profile 的兩個 gNB 都使用同一 TAC，不等於新版 two-TAI scenario；
- 盤點初期 host 只剩約 56 GiB 可用磁碟、swap 幾乎用滿，且 VirtualBox kernel driver
  不可用；舊 VM 清除後雖已回收空間，共用主機的 RAM、disk 與 provider 狀態仍會變動，
  每次新建或長時間運行前都必須重新 preflight。

舊環境仍保有不可遺失的 site-specific 資訊。其 host interface、bridge、IP、route、
VM NIC、MongoDB 與舊 topology 已另行保存於
[local-network-settings-inventory-2026-08-05.md](../reports/local-network-settings-inventory-2026-08-05.md)。
這些資訊是遷移輸入，不是新環境的 public default。

## 3. 設計原則

1. **新的整合邊界**：新建 repository，不複製舊 Git history，也不把舊 dirty tree
   當作基底。
2. **可重現 source identity**：所有直接整合的 source repository 由 parent
   submodule gitlink 固定 exact commit。
3. **類 free5GC superproject 結構**：NF source 集中在 `NFs/`，但不再嵌入一份
   完整 free5GC main repository。
4. **source 不由 guest 決定**：VM provisioning 不自行 clone branch 或 latest
   source；guest 只接收 parent 已固定的 source／artifact 與設定。
5. **free5GC-like config ownership**：`config/` 只保存 NF、RAN、ML 與相關 service
   實際讀取的設定；三 VM topology 與本機差異由 root-level testbed files 管理。
6. **最小但完整**：第一版只支援已驗證的 full-core path；未驗證 NF 不因 free5GC
   main 預設啟動就宣稱支援。
7. **不內嵌 `nwdaf-resources`**：它保留為獨立開發／回歸 repository，不成為新
   infrastructure 的 submodule 或執行前置；只在確認 ownership 後移植必要小工具。
8. **generated state 與 source 分離**：VM disk、container image/layer、binary cache、Python
   environment、MongoDB、ADRF data、log 與 pcap 都不是 submodule。
9. **第一版不支援憑證**：不提供 certificate、TLS 或 OAuth 管理；環境明確限定
   在隔離實驗網路以 HTTP 執行，不暗示 production security readiness。
10. **先保存、再清理舊 VM**：由於本機空間不足，新 VM 建立前先盤點、備份並在使用者
    確認 exact targets 後移除舊 local VM；舊 `5G_Infrastructure` repository、本地腳本
    與 migration inventory 仍保留到新環境通過 fresh-clone E2E。
11. **分離 execution lifecycle**：VM power、guest 5GC／Go services、Host ML containers、
    NWDAF subscriptions 與 traffic/degradation action 分別啟停；任何一層都不隱含重建或
    停止其他層。
12. **GPU 留在 Host**：Python ML backend 以 Docker 使用實體機 GPU，不把 CUDA runtime
    和 NVIDIA device passthrough 塞入一般 Vagrant guest；CPU/GPU device selection 仍由
    component config 控制，不寫死在程式碼。

## 4. Repository 責任與邊界

`5G_NWDAF_Infrastructure` 負責：

- component revision 組合；
- build orchestration 與 artifact identity；
- VM topology、network、single-project Vagrant orchestration 與 Host container placement；
- host-side preflight、VM／service／container／subscription start／stop／status；
- guest-side OS setup、role-local non-ML build、network 與 service lifecycle；
- PyAnLF／PyMTLF image build、GPU runtime gate 與 Host-to-VM endpoint wiring；
- component config 與本機 testbed override；
- public quick start 與後續 E2E bring-up。

它不負責：

- 在 parent repo 直接維護各 NF、NWDAF、PyAnLF 或 PyMTLF 的 feature code；
- 保存 VM image、runtime database、dataset output、log 或 build cache；
- vendor 整個 `nwdaf-resources` 或要求使用者另行 clone 它才能建立三 VM；
- 承諾 production security、HA、capacity benchmark 或真實 application traffic
  performance；
- 自動套用本實驗室的 public interface、IP 或 bridge。
