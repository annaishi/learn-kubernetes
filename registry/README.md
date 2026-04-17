# Kubernetes ローカル開発環境の構築：Ingress & Local Registry 統合編

この演習では、`kind` を使用して、自作 Docker イメージを即座に利用でき、かつブラウザから L7 パスベースでアクセス可能な高度な開発環境を構築しました。

## 1. 構成図


- **Ingress ポート**: `localhost:8080` (HTTP) / `localhost:8443` (HTTPS)
- **Registry ポート**: `localhost:5001` (Host) -> `5000` (Container)

## 2. インフラ構築手順

### ステップ 1：イメージ倉庫（Registry）の起動
```bash
docker run -d --restart=always -p "127.0.0.1:5001:5000" --network bridge --name "kind-registry" registry:2
```

### ステップ 2：クラスターの作成 (`test-config.yaml`)
ポート 80 が他のプロセス（`cloud-provider-kind` 等）に占有されていたため、ホスト側を `8080` に逃がして構築しました。


### ステップ 3：ネットワークの橋渡しと「住所の書き換え」
ここが本演習の最重要ポイントです。

```bash
# 1. Dockerネットワークの結合
docker network connect "kind" "kind-registry"

# 2. 住所の書き換え（hosts.toml の注入）
# クラスター内部の「localhost:5001」という呼び名を、
# 隣のコンテナである「kind-registry:5000」へ転送する設定を強制注入します。
REGISTRY_DIR="/etc/containerd/certs.d/localhost:5001"
for node in $(kind get nodes); do
  docker exec "${node}" mkdir -p "${REGISTRY_DIR}"
  cat <<EOF | docker exec -i "${node}" cp /dev/stdin "${REGISTRY_DIR}/hosts.toml"
[host."http://kind-registry:5000"]
EOF
done

# 3. K8s 側への ConfigMap 適用
# クラスターに「ローカル倉庫が存在すること」を公式に知らせます。
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: local-registry-hosting
  namespace: kube-public
data:
  localRegistryHosting.v1: |
    host: "localhost:5001"
EOF
```

## 3. 疎通テスト（物流ルートの確認）

自作アプリを動かす「最短パイプライン」が機能するか、Nginx を自作イメージに見立ててテストします。

1. **イメージのローカライズ**:
   ```bash
   docker pull nginx:latest
   docker tag nginx:latest localhost:5001/my-nginx:v1
   ```
2. **Push（Mac → 倉庫）**:
   ```bash
   docker push localhost:5001/my-nginx:v1
   ```
3. **Deploy（倉庫 → クラスター）**:
   ```bash
   kubectl run registry-test --image=localhost:5001/my-nginx:v1
   ```
   - **目的**: クラスターが `localhost:5001` という名前を正しく解釈し、イメージを引き抜けるか（Pullできるか）の最終確認。

## 4. トラブルシューティング & 学んだポイント

- **`ErrImagePull` の解決**: 
  クラスター内の `localhost` は「自分自身（Node）」を指す。そのため、`hosts.toml` を書き換えて通信を `kind-registry` コンテナへルーティングさせる必要がある。
- **ポート 80 競合の特定**:
  `sudo lsof -i :80` で犯人（今回は `cloud-provider-kind`）を特定。今回はクラスター側のポートを `8080` にずらすことで平和的に解決した。
- **リソースの贅沢使い**:
  Docker Desktop に RAM 16GB / CPU 8コアを割り当てることで、Kubernetes の重い起動プロセスも余裕を持って動作可能。