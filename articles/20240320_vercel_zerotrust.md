---
title: "Cloudflare Zero Trust と Vercel でお手軽ステージング環境"
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["cloudflare", "vercel"]
published: true
---

# はじめに
Vercel 上にデプロイしている Next.js アプリケーションのステージング環境を Cloudflare Zero Trust でお手軽に作る方法をご紹介します。

## 前提
- チーム開発
- Vercel は Hobby プラン
- Cloudflare の DNS を用いてドメインを管理している

# ドメインの割り当て
まずは、Vercel のプレビュー環境に対してドメインを割り当てます。
通常、プレビュー環境には Vercel Authentication がかかっているため、メンバー以外のアクセスはできませんが、ドメインを割り当てることで誰でもアクセス可能になります。

### ドメインの追加
Vercel のプロジェクト設定 > Domains からプレビューのアクセスに使用するドメインの追加をします。
![Add Domain](/images/20240320_vercel_zerotrust/add_domains.png)

### Git Branch の変更
追加したドメインのEdit から Git Branch を適切に設定し、宛先がプレビュー環境の URL になるように設定します。
![Edit Domain](/images/20240320_vercel_zerotrust/edit_domain.png)

### Cloudflare の DNS 設定
Vercel にドメインを追加した際に表示される DNS 設定を Cloudflare に追加します。
この際、Proxy を**有効**にしてください。
![Vercel DNS](/images/20240320_vercel_zerotrust/dns_cname.png)
![Cloudflare DNS](/images/20240320_vercel_zerotrust/cloudflare_dns.png)

### SSL/TLS 設定
Cloudflare の SSL/TLS 設定で SSL を**フル**に設定します。
![SSL/TLS](/images/20240320_vercel_zerotrust/cloudflare_ssl.png)

### Vercel の Domains から確認
Vercel のプロジェクト設定 > Domains から追加したドメインが表示されていることを確認します。
下記のマークが表示されたら正常です。
![Vercel Domains](/images/20240320_vercel_zerotrust/vercel_check.png)

:::message
ここまで完了したら、追加したドメインでプレビュー環境にアクセスできるようになります。
シークレットモードなどを使用し、誰でもアクセスできるようになっていることを確認してください。
:::

# Cloudflare Zero Trust の設定
このままだと、プレビュー環境に誰でもアクセスできてしまいます。
Cloudflare Zero Trust の Access を使用することで、特定のユーザーのみアクセスできるように設定します。

### 認証方法の設定
Settings > Authentication から Login methods を設定します。
認証方法はなんでも良いですが、GitHub Organization 等がある場合は GitHub を使用すると便利です。
![Add Login Methods](/images/20240320_vercel_zerotrust/add_login_methods.png)

### アプリケーションの追加
Access > Applications からアプリケーションの追加をします。
Application Type は Self-hosted を選択してください。
Policy は Allow を選択し、適切な Include ルールを設定します。
Email や GitHub Organization などを設定するのが良いと思います。

### アプリケーションの追加（その2）
このままだと、Vercel 側で証明書発行に使用する URL までブロックされてしまうため、Vercel の証明書発行に使用する URL を許可するために、アプリケーションをもう一つ追加します。
サブドメイン・ドメインは先ほどと同様に設定した上で、 Path に `/.well-known/acme-challenge/` を設定します。
![Add Application](/images/20240320_vercel_zerotrust/cert.png)
Policy は Bypass を選択し、Include は Everyone を選択します。

:::message
先に設定したアプリケーションのポリシーで Path を除外できれば良いのですが、僕が調べた限りでは見つけられませんでした。
:::

### 確認
設定が済むと下記のようになっているはずです。
![Applications](/images/20240320_vercel_zerotrust/access.png)

# アクセス制限の確認
これで、Vercel のプレビュー環境にアクセスできるユーザーは Cloudflare Zero Trust の設定したユーザーのみになります。
実際にアクセスしてみて、設定したユーザー以外はアクセスできないことを確認してください。

# おわりに
Vercel のプレビュー環境に Cloudflare Zero Trust を使用することで、Hobby プランでも簡単にステージング環境を作ることができました。
