# What Is 5g-viz

## 簡介

`5g-viz` 是一個把 5G testbed 實驗過程轉成「可看、可停、可回放」畫面的觀測層。

它不是用來控制 5GC / NWDAF 的 orchestrator，而是把已經發生的 log、事件與 metrics 轉成比較容易理解的 UI。

## 它在解什麼問題

只直接看 VM log 時，常見問題包括：

- 可觀測訊息很多，但不容易判斷目前哪些 NF 正在互動
- 事件已經發生，但 event 與 traffic chart 之間不容易對齊
- 實驗結束後很難用同一個視角回看整段過程

`5g-viz` 做的事情就是把這些觀測拆成三種畫面：

- topology：把事件轉成節點狀態、邊動畫、pulse、class 變化
- event log：把 parser 產出的結構化事件按時間列出來
- Grafana：把部分 event 進一步投影成 Prometheus metrics，再畫成圖表

## 主畫面與三個觀測面

主畫面不是一張單一的「總圖」，而是三個不同觀測面放在同一頁：

1. topology  
   用來看互動、流程、節點狀態、現在有哪些東西正在發生。

2. event log  
   用來看剛剛到底是哪一種事件發生了，以及事件 payload 長什麼樣。

3. Grafana  
   用來看 traffic prediction、ground truth、deviation、retrain 這類 metrics 層的變化。

這三塊互相關聯，但不是同一份資料的三種樣式。  
後續在模式差異、DVR 與 troubleshooting 上，很多現象都建立在這個前提上。

## 它怎麼把實驗變成畫面

先用高層方式看，可以把整套系統想成下面這條路：

```text
遠端 VM log
  -> 結構化 event
  -> topology / event log
  -> 部分 event 再轉成 metrics
  -> Prometheus / Grafana
```

比較重要的是：

- 所有畫面最早的共同來源是 event
- 不是所有 event 都會進 Grafana
- replay 真正重播的核心資料是 session 目錄裡的 event 與 topology config

## 主要使用情境

### 1. 實驗正在跑時

`live` 模式用於：

- 看 topology 有沒有動
- 看 event log 剛剛發生了什麼
- 看 Grafana 線圖是否持續更新
- 必要時 pause / scrub / play / go live

### 2. 實驗跑完後

`replay` 模式用於：

- 載入一個既有 session
- 從頭或從某個時間點回看 topology 與 event log
- 讓 Grafana 對應到那次錄製的資料
- 在播放時，讓 chart 看起來像「歷史資料正在即時播放」

## 它不是什麼

為了避免一開始就誤解，下面幾件事很重要：

- 它不直接啟動或編排 5GC / NWDAF 元件
- 它不是自製 chart engine；圖表仍然交給 Grafana
- 它不是單純的 log viewer；它會把 log 轉成 event、state 與 metrics
- 它也不是把 Prometheus TSDB 當成唯一錄製格式；可攜 replay 依賴的是 session 目錄

## 核心心智模型

把 `5g-viz` 想成：

> 一個把 5G testbed 實驗過程拆成 topology、event、metrics 三層觀測，並支援 live 與 replay 切換的觀測與重播介面。

後續閱讀：[Live Vs Replay](./live-vs-replay.md)
