# Features

本目錄收錄跨層功能流程文件，用來描述單一功能如何穿越 backend、frontend、Grafana 與 DVR。

適合的情況包括：

- 需要追某條 feature flow 的端到端路徑
- 需要把 human-facing explanation 往下連到具體跨層案例

這一層比較像具體案例，不是第一層閱讀入口。

主要文件：

- [traffic-chart.md](traffic-chart.md)：NWDAF 流量圖表從 event、metrics 到 Grafana iframe 的完整資料路徑
- [subscription-chain.md](subscription-chain.md)：Consumer、NWDAF、SMF、UPF、Consumer 之間的訂閱與通知鏈路
- [nwdaf-ml-cycle.md](nwdaf-ml-cycle.md)：UPF volume、AnLF 推論、MTLF 重訓、ADRF 取數到 model swap 的完整循環
