---
layout: ../../layouts/MarkdownPostLayout.astro
title: "proxmoxのWeb GUIにアクセスできない話"
description: "関数・ループ・配列など、競プロでよく使われるより実践的なC++の使い方を紹介します。"
date: "2024-03-24"
authors: ["kasa021"]
tags: ["proxmox", "web gui", "ネットワーク"]
---

## はじめに
自宅ラボに **Proxmox VE 8.3** を新規インストールしたところ、管理 GUI（https://\<IP\>:8006/）へブラウザからアクセスできずにハマりました。備忘録として、

* 発生した症状
* 原因の切り分けと特定
* 最終的な解決策
* 調査・修復に使ったコマンド一覧

をブログ形式でまとめます。

---

## 環境
| 役割 | デバイス | 接続方法 | IP / サブネット |
|------|----------|----------|-----------------|
| Proxmox ホスト | 自作 PC | 有線 LAN | **192.168.2.150/24**（当初設定） |
| MacBook Pro | macOS 14 | 有線 LAN | 192.168.0.100/24 |
| ホームゲートウェイ | auひかり HGW | ルータ | **192.168.0.1/24**（DHCP 128 台） |

> **ポイント** : Proxmox だけが 192.168.2.0/24、ほかは 192.168.0.0/24 というズレが後のトラブルを招きました。

---

## 症状
* ブラウザで `https://192.168.2.150:8006/` にアクセスすると **ERR_CONNECTION_TIMED_OUT**。
* Mac から `ping 192.168.2.150` → `No route to host`。
* Proxmox 側では `curl -k https://127.0.0.1:8006/` が **HTTP 200** を返し、GUI サービス自体は稼働。****

---

## 調査の流れ
1. **Proxmox 側のサービス確認**  
   `ss -lntp | grep 8006` → `*:8006` で LISTEN。問題なし。
2. **ARP テーブルを確認**  
   Mac で `arp -a | grep 192.168.2.150` → `(incomplete)`。
3. **tcpdump で ARP パケットを確認**  
   Proxmox から ARP 要求は出るが応答が来ない。
4. **ネットワーク図を見直し**  
   HGW は 192.168.0.1/24。Proxmox だけ別サブネットにいることに気付く。

---

## 原因
ARP は **同一 IP サブネット** 内でのみ動作するため、

* Proxmox: 192.168.2.150/24
* Mac & HGW: 192.168.0.0/24

ではレイヤ 2 で到達できず、結果としてタイムアウトしていました。

---

## 解決策
### 1. 一時的に IP を変更して疎通テスト
```bash
# 旧 IP を外す
ip addr del 192.168.2.150/24 dev vmbr0
# 新 IP を付与（DHCP プール外）
ip addr add 192.168.0.200/24 dev vmbr0
# デフォルト GW を HGW に
ip route replace default via 192.168.0.1 dev vmbr0
```
→ Mac から ping / nc が通り、GUI 表示も成功。

### 2. 恒久設定
`/etc/network/interfaces` を以下のように書き換え、
```ini
auto vmbr0
iface vmbr0 inet static
    address 192.168.0.150/24
    gateway 192.168.0.1
    bridge-ports enp3s0
    bridge-stp off
    bridge-fd 0
```
`ifreload -a` で即反映。

---

## コマンド一覧とサンプル出力
| フェーズ | コマンド | 目的 / 期待出力例 |
|-----------|----------|-------------------|
| サービス確認 | `ss -lntp \| grep 8006` | `LISTEN 0 4096 *:8006 ... pveproxy` |
| 内部 HTTP テスト | `curl -k https://127.0.0.1:8006/ -o /dev/null -w "%{http_code}\n"` | `200` |
| IP 確認 | `ip -4 addr show vmbr0` | `inet 192.168.2.150/24` |
| ルート確認 | `ip route show` | `default via 192.168.2.1 dev vmbr0` |
| ARP テーブル | `ip neigh show dev vmbr0` | (空 or FAILED) |
| ARP キャプチャ | `tcpdump -eni vmbr0 arp` | `Who-has ...` のみで `is-at` 無し |
| Mac 側ルート | `route -n get default` | `gateway: 192.168.0.1` |
| IP 変更（暫定） | `ip addr add/del`, `ip route replace` | - |
| 設定反映 | `ifreload -a` | `reload interfaces: vmbr0 ... done` |

---

## まとめ
* **症状** : GUI にアクセスできない／ping が届かない。
* **原因** : Proxmox だけ別サブネット (192.168.2.0/24)。ARP が解決できず通信不可。
* **対処** : Proxmox の管理 IP をホームゲートウェイと同じ 192.168.0.0/24 に変更。

同じように「ローカル機器同士なのに繋がらない」ときは、まず **IP サブネットの一致** と **ARP テーブル** を疑うと解決が早いです。

---

