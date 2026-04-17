
# Kubernetes LoadBalancer 演習：外部 IP による負荷分散の構築

`kind` 環境において、通常はクラウドプロバイダー（AWS 等）が必要な `type: LoadBalancer` を、`cloud-provider-kind` を使ってローカル環境で再現した記録です。

## 1. 構成イメージ


通信の流れ：
`Mac (curl)` -> `External IP (172.19.0.6)` -> `Service (LoadBalancer)` -> `Pod (foo or bar)`

## 2. 準備：Cloud Provider KIND の起動

kind に外部 IP を払い出す機能を追加するため、Mac 上でツールを起動します。

インストール
```bash
brew install cloud-provider-kind
```

実行
```bash
# 管理者権限が必要
sudo cloud-provider-kind
```
※このプロセスは実行したまま、別のターミナルで作業を続けます。

## 3. アプリケーションと Service のデプロイ

2 つの Pod に共通のラベル（`app: http-echo`）を付け、1 つの Service でそれらを束ねます。

**実行コマンド:**
```bash
kubectl apply -f lb-apps.yaml
```

## 4. 動作確認

`EXTERNAL-IP` が割り当てられたことを確認し、アクセスを繰り返します。

```bash
kubectl get svc foo-service
```

| テスト内容 | コマンド | 期待される結果 |
| :--- | :--- | :--- |
| **負荷分散の確認** | `curl http://[EXTERNAL-IP]:5678` | `foo-app` と `bar-app` が交互またはランダムに返る |



## 5. 学んだ重要概念

- **Service Selector**: 名前が違っても、特定の「ラベル」を持つ Pod をグループ化して通信を振り分けることができる。
- **EXTERNAL-IP**: `type: LoadBalancer` を指定すると割り振られる、クラスター外からアクセスするための住所。
- **cloud-provider-kind**: 本来クラウド環境で提供される LB 機能をローカルの Docker ネットワーク上でシミュレートするツール。
- **Ingress との違い**:
  - **Ingress**: L7 層。URL パス（`/foo`）を見て、**宛先 Service を切り替える**。
  - **LoadBalancer**: L4 層。1 つの窓口で、**背後の Pod たちに均等に** 負荷を分散する。

## 6. トラブルシューティングメモ
- `EXTERNAL-IP` が `<pending>` のままの場合、`cloud-provider-kind` が `sudo` で実行されているか、別ターミナルで生きているかを確認する。
