# Kubernetes Ingress 演習：パスベースルーティングの構築

`kind`（Kubernetes in Docker）環境を使用して、1つの窓口（ポート80）からリクエストパスに応じて異なるアプリ（Pod）へ振り分ける「パスベースルーティング」を構築した記録です。

## 1. 構成イメージ


通信の流れ：
`Mac (Browser/curl)` -> `kind Node (Port 80)` -> `Ingress Controller (Nginx)` -> `Service` -> `Pod`

## 2. インフラ準備 (kind クラスター作成)

Ingress をローカルで動作させるには、Node 起動時に「ポートマッピング」と「特定のラベル」を付与する必要があります。

**実行コマンド:**
```bash
kind create cluster --config kind-ingress-config.yaml
```

## 3. Ingress Controller (Nginx) の導入

Ingress 設定（ルール）を実際に処理する「実体」をインストールします。

```bash
kubectl apply -f [https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml](https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml)
```
※ `kubectl get pods -n ingress-nginx` で Pod が `Running` になるまで待機します。

## 4. アプリケーションのデプロイ

`agnhost` イメージを使用して、自身のホスト名を返す 2 つのアプリ（foo/bar）と、それらを振り分ける Ingress ルールを定義します。

**ファイル名: `my-app.yaml`**
- **Pod / Service**: `foo` 用と `bar` 用の 2 セットを作成。
- **Ingress**: `/foo` を `foo-service` へ、`/bar` を `bar-service` へルーティング。

**実行コマンド:**
```bash
kubectl apply -f my-app.yaml
```

## 5. 動作確認

| テスト内容 | コマンド | 期待される結果 |
| :--- | :--- | :--- |
| **foo へのアクセス** | `curl http://localhost/foo` | `foo-app` |
| **bar へのアクセス** | `curl http://localhost/bar` | `bar-app` |

## 6. 学んだ重要キーワード

- **Ingress**: ALB のような L7 ロードバランサー。URL パスやドメインで振り分ける。
- **Ingress Controller**: Ingress ルールを読み取って実際に通信を制御するプロキシ（今回は Nginx）。
- **kind extraPortMappings**: Docker コンテナの外（Mac）からコンテナ内の Node へ通信を通すための設定。
- **kube-proxy**: Node 内で Service への通信を適切に Pod へ振り分ける交通整理役。
- **CNI**: Pod 同士の通信経路を作るネットワークプラグイン。
- **kubeadm**: クラスターの初期化と Node のセットアップを自動化するツール。
```