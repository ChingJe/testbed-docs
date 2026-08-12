# PLMN Single-Source Config Regression

日期：2026-08-12

## 結論

`5G_NWDAF_Infrastructure/testbed.yaml:mobileNetwork.plmn` 已成為唯一 MCC/MNC
來源。只修改該欄位後，renderer 會同步產生 NF/RAN PLMN、TAI、15-digit IMSI
SUPI、Internal Group ID、UE routing、Consumer target 與 subscriber/group
fixtures。既有 `466/92` default config 通過新版 checker，實際 identity 未改變。

GPSI/MSISDN 明確保留為獨立 fixture identity。即使測試 PLMN 改為 `001/01` 或
`001/001`，六筆 `msisdn-4669200001` 至 `msisdn-4669200006` 仍保持原值；這是
刻意的 contract，不是漏套 PLMN。

本輪沒有修改 NF、ML、UERANSIM 或 WebConsole submodule source，也沒有啟動 VM、
container 或 Guest process。

## Topology contract

- MCC 必須為 3 位十進位數字；MNC 必須為 2 或 3 位十進位數字。
- 每個 Path 只保存 TAC，不再保存重複的 PLMN。
- 每個 Path 保存三個唯一正整數 `subscriberNumbers`。MSIN 依 MCC/MNC 長度補零，
  使 IMSI 固定為 15 digits。
- Internal Group 只保存 8-hex Group Service ID 與 2–20-hex、完整 octet 的 Local
  Group ID；完整值由 service、MCC、MNC、local 四段組成。
- topology 若重新加入完整 `internalGroupId`、`tai.plmn`、完整 `ues` 或 Consumer
  group ID，checker 會視為第二來源並拒絕。

## Render 與 checker coverage

renderer 與 checker 共用 `configlib.resolve_mobile_identities()`，涵蓋：

- NRF `DefaultPlmnId`；
- AUSF、NSSF、AMF、SMF 與 ADRF 的 PLMN／S-NSSAI／TAI 欄位；
- UDM Internal Group range；
- SMF 的兩個 UPF TAI；
- NWDAF-A/B tracking TAI；
- gNB/UE MCC、MNC、TAC 與 SUPI；
- UE routing 的 Path A/B member list；
- Consumer PLMN、Internal Group 與 TAC；
- subscriber fixture 的 serving PLMN、SUPI、S-NSSAI、DNN；
- group fixture 的 Internal Group ID 與 member SUPI。

dataset resolver 也改用衍生後的 Path SUPI 數量計算 UE IP，不再依賴 topology 中
已移除的完整 SUPI list。

## Positive regression

`scripts/host/mobile-identity-smoke.py` 在 temporary directory 建立兩套 topology 與
完整 config，逐套執行 renderer、checker 及 identity assertions：

| Case | PLMN | 第一筆 SUPI | Internal Group ID |
|---|---|---|---|
| 2-digit MNC | `001/01` | `imsi-001010000000001` | `00000001-001-01-01` |
| 3-digit MNC | `001/001` | `imsi-001001000000001` | `00000001-001-001-01` |

兩案皆通過，且六筆 GPSI 與 baseline 完全相同。

## Negative regression

`config-contract-smoke.py` 分別製造以下單檔 drift，皆由 checker 拒絕：

- UE1 衍生 SUPI；
- Path A UE-routing members；
- NSSF supported PLMN；
- SMF UPF-A TAI PLMN；
- Consumer Internal Group ID。

因此 config set 無法只改動其中一份 identity 後仍通過啟動前驗證。

## Commands and results

```text
python3 scripts/host/config-check.py --testbed testbed.yaml --config-dir config/default
OK

python3 scripts/host/mobile-identity-smoke.py
OK case=mnc-2 ...
OK case=mnc-3 ...

python3 scripts/host/config-contract-smoke.py --testbed testbed.yaml --config-dir config/default
29 negative cases rejected as expected

make test
shell/Python syntax, config contracts, mobile identities, WebConsole disabled/enabled
config, network aliases, deterministic dataset, Compose baseline/CPU config: PASS
Vagrant home permission probe: blocked only by the managed sandbox read-only home

vagrant validate
Vagrantfile validated successfully.
```

最後一項是在允許 Vagrant 使用正常 `~/.vagrant.d` 狀態目錄後獨立補跑；它只驗證
definition，沒有執行 `vagrant up` 或改動 VM lifecycle。
