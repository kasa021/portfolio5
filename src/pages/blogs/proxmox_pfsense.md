---
layout: ../../layouts/MarkdownPostLayout.astro
title: Proxmox + pfSense で自宅ネットワークにルータを構築する手順
description: "Proxmox + pfSense で自宅ネットワークにルータを構築する手順"
date: "2024/04/26"
authors: ["kasa021"]
tags: ["proxmox", "pfSense", "ネットワーク"]
---
このドキュメントでは、Proxmox上にpfSenseを構築して、自宅にプライベートネットワーク（LAN）を作り、そこからインターネットにアクセスできるようにした一連の手順をまとめます。

---

## ⚙️ 構成概要

```
[Internet] ⇔ [Home Gateway (ISP)] ⇔ [mini PC (Proxmox)]
                                         └─ pfSense VM
                                              ├─ WAN: 192.168.0.x/24 (DHCP)
                                              └─ LAN: 192.168.1.1/24 (Static)
                                                         ⇓
                                                      Switch ⇔ PC1
```

---

## ✅ ステップバイステップ

### ステップ 1: Proxmox のネットワークブリッジ設定

`/etc/network/interfaces` にて、以下のように設定:

```ini
auto vmbr0
iface vmbr0 inet static
    address 192.168.0.xxx/24
    gateway 192.168.0.1
    bridge-ports enp3s0
    bridge-stp off
    bridge-fd 0

auto vmbr1
iface vmbr1 inet static
    address 192.168.1.2/24
    bridge-ports enp2s0
    bridge-stp off
    bridge-fd 0
```

- `vmbr0` はホームゲートウェイへ接続 (WAN)
    
- `vmbr1` はLAN用 (スイッチ接続)
    

### ステップ 2: pfSense VM のネットワークデバイス構成

Proxmox上で pfSense VM に以下のように設定:

|デバイス|ブリッジ|用途|
|---|---|---|
|Net0|vmbr0|WAN|
|Net1|vmbr1|LAN|

pfSense 起動後、以下のようになっていることを確認:

- WAN (vtnet0): 192.168.0.x (DHCP)
    
- LAN (vtnet1): 192.168.1.1 (Static IP)
    

### ステップ 3: pfSense WebGUIへアクセス

- PC1 から [https://192.168.1.1](https://192.168.1.1/) へアクセス
    
- 初期ユーザー: `admin` / パスワード: `pfsense`
    

セットアップウィザードはスキップしてOK

### ステップ 4: DHCP サーバの設定

WebGUIから:

```
Services > DHCP Server > LAN
```

- DHCP: Enable にチェック
    
- Range: `192.168.1.100` ～ `192.168.1.200`
    
- DNS Server: `192.168.1.1`, `1.1.1.1`
    

### ステップ 5: NAT の確認

WebGUIから:

```
Firewall > NAT > Outbound
```

- Mode: `Automatic Outbound NAT rule generation` を選択
    

---

## ✅ 動作確認

- PC1 に `192.168.1.100` が割り当てられる
    
- `ping 192.168.1.1` → OK
    
- `ping 8.8.8.8` → OK
    
- `curl http://example.com` → OK
    

---

## 🎉 完成したネットワーク構成

- pfSense が NAT ルータとして機能
    
- LAN配下のPCがインターネットへアクセス可
    
- Proxmox上に追加でVM1/VM2もLANに参加可
    

---

## ☕️ 仕上げについて

- pfSense admin パスワードは後で変更すること
    
- PC1には勝手にIPが割り当てられる
    
- DNS・NATも正常に動作中
    

---
