---
layout: ../../layouts/MarkdownPostLayout.astro
title: Proxmox VE の apt-get update で 401 Unauthorized エラーを解決した話
description: "Proxmox VE の apt-get update で 401 Unauthorized エラーを解決した話"
date: "2024-04-10"
authors: ["kasa021"]
tags: ["proxmox", "apt-get", "エラー"]
---

## はじめに

Proxmox VE を利用している際に、`apt-get update` を実行すると Enterprise リポジトリからのパッケージ取得時に 401 Unauthorized エラーが発生し、更新が途中で止まってしまいました。  
本記事では、発生した症状、原因の切り分け、最終的な解決策、そして調査・修正に使用したコマンドをブログ形式でまとめます。

---

## 環境

| 役割               | デバイス              | 接続方法    | IP / サブネット                  |
| ---------------- | ----------------- | ------- | ----------------------------- |
| Proxmox ホスト   | サーバ（例：自作 PC） | 有線 LAN | **例: 192.168.1.100/24**       |
| 更新サーバ       | Debian 12 Bookworm  | 有線 LAN | 同一ネットワーク内              |

> **ポイント**: Enterprise リポジトリは有償サブスクリプションキーが必要ですが、今回はキーが未登録だったためにエラーが発生しました。

---

## 症状

- `apt-get update` を実行すると、Enterprise リポジトリからの更新時に **401 Unauthorized** エラーが表示されました。
- 出力例:```
```
  Err: https://enterprise.proxmox.com/debian/pve bookworm InRelease
    401  Unauthorized [IP: 51.79.228.122 443]
  E: The repository 'https://enterprise.proxmox.com/debian/pve bookworm InRelease' is not signed.
```

**調査の流れ**

1. **パッケージリストの更新確認**
apt-get update を実行して、どのリポジトリでエラーが発生しているか確認しました。

1. **リポジトリ設定の確認**
/etc/apt/sources.list および /etc/apt/sources.list.d/ 内のファイルをチェックし、Enterprise リポジトリが有効になっていることを確認しました。

1. **Enterprise リポジトリの仕様確認**
Proxmox の公式ドキュメントから、Enterprise リポジトリは有償サブスクリプションキーが必要であることを再確認しました。

1. **No‑Subscription リポジトリの利用検討**
検証環境では無償の No‑Subscription リポジトリを利用することで、問題を回避できることを確認しました。

**原因**

Enterprise リポジトリからのパッケージ取得時に、サブスクリプションキーが未登録だったため、認証エラー（401 Unauthorized）が発生しました。

その結果、apt-get update が途中で止まってしまいました。