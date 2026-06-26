---
title: "iPhoneからMacのClaude Codeを操作する — Tailscale + Agent Viewで複数セッション管理"
emoji: "📱"
type: "tech"
topics: ["claudecode", "tailscale", "ssh", "ios"]
published: true
---

:::message
**想定読者**: Claude Code を日常的に使っている Mac ユーザーで、外出先や別の部屋からも iPhone で進捗確認・タスク投入をしたい方。
:::

## この記事で解決すること

Claude Code にはスマホから操作できる公式の **Remote Control** 機能がある。Remote Control には主に3つの始め方がある。

| モード | 複数セッション | ローカル対話 | 既存セッション引き継ぎ |
|---|---|---|---|
| `claude --remote-control` | 1 プロセスにつき 1 セッション | 可能 | 起動時のみ |
| セッション内の `/remote-control` | 1 プロセスにつき 1 セッション | 可能 | 可能 |
| `claude remote-control`（Server Mode） | 複数可（デフォルト capacity 32） | 不可（サーバー専用） | 不可 |

Remote Control は「1つの実行中セッションを別デバイスから継続する」用途には強い一方、**ローカルの1画面で複数のバックグラウンドセッションを一覧管理し、必要に応じて attach / detach する**用途は **Agent View (`claude agents`)** の方が合っている。

Agent View なら **ローカル対話と複数セッション管理を両立** できる。Mac で普通に Claude Code を使い、`←` キーひとつでセッションをバックグラウンドに移して一覧管理。iPhone からも同じ Agent View に SSH で入れる。

この記事では **Tailscale + Termius で iPhone から Mac に SSH 接続し、Agent View で複数セッションを管理する構成** を紹介する。

### Remote Control と Agent View の比較

| | Remote Control | SSH + Agent View |
|---|---|---|
| 複数セッションの一覧管理 | Server Mode のみ（ローカル対話不可） | **可能（ローカル対話と両立）** |
| 既存セッションの取り込み | `/remote-control` で 1 セッション | **`/bg` で複数セッション** |
| セットアップ | ほぼゼロ | 初回のみ 15 分程度 |
| 認証方式 | OAuth 必須 | 制限なし |

:::message
この記事は Claude Code **v2.1.153** 時点の情報。`claude agents` は v2.1.139 以降で利用可能。
:::

## 構成の全体像

```
iPhone (Termius)
    │
    │ Tailscale VPN (100.x.x.x)
    │
    ▼
Mac (claude agents)
    ├── Session A: プロジェクトXのバグ修正 [working]
    ├── Session B: PRレビュー対応 [awaiting input]
    └── Session C: テスト追加 [completed]
```

必要なのは3つだけ。

| コンポーネント | 役割 | 費用 |
|---|---|---|
| **Tailscale** | WireGuard ベースの VPN。NAT 越えを自動で処理 | 個人利用無料 |
| **Termius** | iPhone 向け SSH クライアント | 無料版で十分 |
| **`claude agents`** | Claude Code 組み込みの Agent View | Claude Code に付属 |

## セットアップ

### Mac 側

#### 1. Tailscale のインストール

Mac App Store から [Tailscale](https://apps.apple.com/app/tailscale/id1475387142) をインストールしてログイン。

App Store 版は CLI が PATH に入らない。必要に応じて wrapper を作成する。

```bash
mkdir -p ~/bin
cat > ~/bin/tailscale << 'SCRIPT'
#!/bin/zsh
exec /Applications/Tailscale.app/Contents/MacOS/Tailscale "$@"
SCRIPT
chmod +x ~/bin/tailscale
```

`~/bin` が PATH に入っていれば `tailscale status` で動作確認できる。

#### 2. SSH の有効化

**システム設定 → 一般 → 共有 → リモートログイン** を ON にする。

SSH 鍵認証の設定は iPhone 側のセットアップ後に行う（iPhone で鍵を作成し、その公開鍵を Mac に登録する流れ）。

Tailscale IP はメニューバーの Tailscale アイコン、または `tailscale ip` で確認できる。

### iPhone 側

#### 1. Tailscale

App Store から **Tailscale** をインストールし、Mac と同じアカウントでログイン。両デバイスが「Connected」になれば OK。

#### 2. Termius

App Store から **Termius** をインストール（無料版で十分。Pro への課金は Skip）。

#### 3. SSH 鍵の作成

パスワード認証でも接続は可能だが、セキュリティを考慮すると鍵認証が望ましい。

Termius で鍵を作成する手順:

1. 画面下部の **Keychain** タブを開く
2. **+** → **Generate Key** → **ED25519** を選択
3. 鍵名を設定して保存（例: `mac-remote`）
4. 作成した鍵の公開鍵をコピー

コピーした公開鍵を Mac の `authorized_keys` に追加する。

```bash
# Mac 側で実行
mkdir -p ~/.ssh
chmod 700 ~/.ssh
printf '%s\n' 'ssh-ed25519 AAAA... termius-iphone' >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

（`ssh-ed25519 AAAA...` の部分は Termius で生成した公開鍵に置き換える）

#### 4. ホストの追加

Termius でホストを追加する。

- **Hostname**: Mac の Tailscale IP（`100.x.x.x`）
- **Username**: Mac のユーザー名
- **SSH ID, Key, Certificate, FIDO2**: 作成した鍵を選択

ホストをタップして接続し、シェルのプロンプトが表示されれば成功。

## 使い方

### Agent View を起動する

iPhone の Termius から SSH 接続したら、以下を実行する。

```bash
claude agents
```

初回起動時はバックグラウンドセッションが空のため、下部の入力欄のみ表示される。Mac 側で `/bg` を使って既存セッションを移すか、入力欄からタスクを dispatch する。

セッションがある状態では、以下のような一覧が表示される。

```
Claude Code v2.1.153
Sonnet 4.6 · /Users/you
2 awaiting input · 1 working · 1 completed

  * my-project@bugfix            /my-project       12m  awaiting input
    api-server@review            /api-server        3m  working
    docs@update                  /docs              1h  completed

> Start a task in the background
```

#### 主な操作

| 操作 | 説明 |
|---|---|
| 下部入力欄にタスクを入力 | 新規バックグラウンドセッションを dispatch |
| `Enter` | 選択中のセッションを開く（attach） |
| `Space` | セッションの出力を確認する（peek） |

:::message alert
下部の入力欄は **新規セッションの dispatch 用**。既存セッションに返信したい場合は、そのセッションを選んで `Enter`（attach）または `Space`（peek）から操作する。
:::

### Mac の既存セッションを Agent View に移す

Mac のターミナルで既に Claude Code を動かしている場合、空のプロンプトで `←` キーを押すとバックグラウンドに移行する。`/bg` コマンドでも同じ。

バックグラウンドに移ったセッションは Agent View に表示され、iPhone から操作できるようになる。

### 典型的なワークフロー

1. **Mac で作業開始** — 通常通り `claude` を起動して作業
2. **離席時に `←`** — 空のプロンプトで左矢印キーを押すだけ。セッションがバックグラウンド化され、そのまま Agent View が開く
3. **iPhone から `claude agents`** — 進捗確認、追加タスク dispatch
4. **Mac に戻ったら `claude agents`** — セッションを選んで attach、作業再開

Mac と iPhone のどちらから `claude agents` を開いても、同じセッション一覧が見える。

## 注意点

### SSH 切断時の挙動

Agent View を表示していた SSH セッションが切断されても、バックグラウンドで動作中の Claude セッションには影響しない。再接続後に `claude agents` を実行すれば、同じセッション一覧が表示される。

### セキュリティ

この構成は個人の開発環境を想定している。業務利用の場合は所属組織のセキュリティポリシーを確認すること。

最低限、以下は対応しておいた方がいい。

- **SSH 鍵認証を使う**（パスワード認証よりも安全）
- **SSH 鍵にパスフレーズを設定する**（iPhone 紛失時の防御）
- **Tailscale の ACL を確認する**（不要なデバイスからのアクセスを制限）
- **SSH をインターネットに直接公開しない**（ルーターで 22 番ポートを開けず、Tailscale IP 宛てに接続する）
- **Mac のスリープ設定を確認する**（スリープ中は SSH 接続できない場合がある）

### 利用枠の消費

Agent View から複数セッションを並列で動かすと、各セッションがそれぞれ利用枠を消費する。同時に走らせるセッション数は意識しておくこと。

## トラブルシューティング

| 症状 | 対処 |
|---|---|
| SSH 接続できない | Tailscale が両デバイスで「Connected」か確認 |
| 認証エラー | `authorized_keys` のパーミッション（`chmod 600`）と鍵の対応を確認 |
| Agent View にセッションが表示されない | セッション内で `←` または `/bg` を実行したか確認 |
| 遅延が大きい | `tailscale netcheck` で DERP リレー経由か確認。同一 LAN なら直接接続になるはず |
| Mac がスリープで応答しない | システム設定でスリープを無効化、または `caffeinate` コマンドで一時的に防止 |

## まとめ

Remote Control は手軽だが、ローカル対話と複数セッション管理を両立するなら Agent View が向いている。Tailscale + Termius で SSH 接続を確保すれば、iPhone からも `claude agents` と打つだけ。初回セットアップは15分程度で済む。

## 関連リソース

- [Claude Code — Agent View](https://docs.anthropic.com/en/docs/claude-code/agent-view)
- [Claude Code — Remote Control](https://docs.anthropic.com/en/docs/claude-code/remote-control)
- [Tailscale — Getting Started](https://tailscale.com/kb/1017/install)
- [Termius — SSH Key の作成](https://support.termius.com/hc/en-us/articles/4402386576793-Generating-SSH-keys-in-Termius)
