# manage-dhcp-block-R5

DHCP の有効/無効状態に応じて、起動時に `ebtables` のルールを自動設定するスクリプトです。

Ubuntu 系で `netplan` を利用している環境を想定しています。

---

## 概要

本パッケージは、

* `eth0` の DHCP が有効な場合

  * DHCP パケットを許可
* `eth0` の DHCP が無効な場合

  * DHCP パケット (UDP 67/68) を `ebtables` により遮断

を自動的に行います。

設定状態は

```
/etc/netplan/50-cloud-init.yaml
```

内の

```yaml
eth0:
  dhcp4: true
```

の値から判定されます。

---

## ファイル構成

```
manage-dhcp-block-R5/
├── install.sh
└── setup-ebtables.sh
```

### install.sh

必要パッケージの導入と設定スクリプトの実行を行います。

### setup-ebtables.sh

`/etc/rc.local` に DHCP ガード処理を追加します。

起動時に DHCP の設定状態を確認し、必要に応じて `ebtables` のルールを追加または削除します。

---

## 動作仕様

### DHCP 有効時

netplan:

```yaml
eth0:
  dhcp4: true
```

起動時に

```bash
ebtables -D FORWARD -i eth0 -p IPv4 --ip-proto udp --ip-dport 67:68 -j DROP
```

を実行し、DHCP 通信を許可します。

syslog には

```
DHCP ALLOW on eth0
```

が記録されます。

---

### DHCP 無効時

netplan:

```yaml
eth0:
  dhcp4: false
```

起動時に

```bash
ebtables -A FORWARD -i eth0 -p IPv4 --ip-proto udp --ip-dport 67:68 -j DROP
```

を実行し、DHCP パケットを遮断します。

syslog には

```
DHCP BLOCK on eth0
```

が記録されます。

---

## インストール

### 1. パッケージ展開

```bash
tar xvf manage-dhcp-block-R5.tar.gz
cd manage-dhcp-block-R5
```

### 2. インストール実行

```bash
chmod +x install.sh setup-ebtables.sh

./install.sh
```

内部で以下が実行されます。

```bash
sudo apt update
sudo apt install -y ebtables
sudo ./setup-ebtables.sh
```

---

## rc.local のバックアップ

実行時に現在の

```
/etc/rc.local
```

を

```
/etc/rc.local.bak.YYYYMMDD-HHMMSS
```

としてバックアップします。

例：

```
/etc/rc.local.bak.20260618-093000
```

---

## ログ確認

ログは syslog に記録されます。

確認例：

```bash
grep dhcp-guard /var/log/syslog
```

表示例：

```
dhcp-guard: DHCP BLOCK on eth0
dhcp-guard: DHCP ALLOW on eth0
```

---

## 注意事項

* root 権限（sudo）が必要です。
* Ubuntu + netplan 環境を前提としています。
* DHCP 判定対象インターフェースは

```bash
IFACE="eth0"
```

に固定されています。

変更する場合は `setup-ebtables.sh` 内を編集してください。

* `setup-ebtables.sh` は同じ処理が既に挿入されている場合、重複して追加しません。

---

## 想定用途

* Raspberry Pi
* Ubuntu Server
* ブリッジ構成で DHCP パケットの流入を制御したい環境
* DHCP サーバや誤配布を防止したいネットワーク
* 固定 IP 運用時に不要な DHCP パケットを遮断したい環境

---

## ライセンス

自由に改変・再配布可能です。
利用に伴う損害について作者は責任を負いません。

