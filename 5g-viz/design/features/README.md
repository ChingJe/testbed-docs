# Features

本目錄收錄跨層功能流程文件，用來描述單一功能如何穿越 backend、frontend、Grafana 與 DVR。

主要文件：

- [traffic-chart.md](traffic-chart.md)：NWDAF 流量圖表從 event、metrics 到 Grafana iframe 的完整資料路徑
- [subscription-chain.md](subscription-chain.md)：Consumer、NWDAF、SMF、UPF、Consumer 之間的訂閱與通知鏈路
- [nwdaf-ml-cycle.md](nwdaf-ml-cycle.md)：UPF volume、AnLF 推論、MTLF 重訓、ADRF 取數到 model swap 的完整循環
