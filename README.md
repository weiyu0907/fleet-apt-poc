# Fleet APT 分階段派送 PoC

用 aptly + nginx + unattended-upgrades 建立 Ring-based 套件派送，
在三台 Incus container 上完成九個 Gate 的驗證。

## 架構

    repo     10.30.1.10   aptly + nginx (Ring ACL by IP)
    canary   10.30.1.11   Ring 1, timer 15min
    prod-a   10.30.1.12   Ring 2, timer 60min

版本決策集中在 repo 端的 `aptly publish switch`。
Ring 機器不下版本指令，由 Pin-Priority 1001 決定候選版本、timer 定期套用。

## 目錄

- `repo-server/` — nginx Ring ACL、假套件打包腳本
- `ring-client/` — sources、pin、APT hook、systemd unit、三個驗證腳本
- `docs/` — 驗證結論

## 主要發現

1. postinst 失敗會癱瘓整台機器的套件系統，`dpkg --configure -a` 是死循環
2. 安靜的失敗只有版本比對抓得到（apt/dpkg/systemd 三者都報正常）
3. `apt-get -y full-upgrade` 拒絕降版，須加 `--allow-downgrades`
4. `apt-get update` 遇 BADSIG 回傳 0，須用 `--error-on=any`
5. apt lists 快取會讓錯誤狀態延續，檢查前須清快取
6. unattended-upgrades 會執行降版，不需額外選項，與手動 apt 行為相反
7. 降版在 u-u 的 INFO log 裡與升級完全同形
8. apt-daily-upgrade 有天級節流，被跳過時仍產生假陽性
9. aptly 的信任根 `trustedkeys.gpg` 與系統 apt 完全分開

## 部署腳本骨架

    rm -rf /var/lib/apt/lists/* /var/lib/apt/lists/partial/*
    apt-get update --error-on=any
    apt-get -y --allow-downgrades full-upgrade
    /usr/local/bin/fleet-healthcheck

四行分別對應發現 5、4、3、2。

## 狀態

PoC 階段。IP 與套件名稱需依實際環境調整。
金鑰為 PoC 用法（`%no-protection`），正式環境須有 passphrase。
