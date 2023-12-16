# IPFS-CLUSTER

## 說明

- 由於 helm-ipfs-cluster 官方沒有做 helm repo，因此這裡直接幹下來用
  - 難用，跟他說再見
- ipfs-operator
  - 還在開發中，但至少 7 個月沒更新了，因算能說開發穩定（死一半）
  - 但不用自己管狀態
  - ~~沒有做 gateway, 哭了~~

## ipfs-operator - Setup

- 安裝

```bash
cd ipfs-operator
kubectl apply -k overlays/production
```

- 頗 Hack 的方式去改設定 (因為設定是由 operator 產的，且本身不給改設定)

```bash
# 原本預設設定是 tcp, 但 operator 開的 `pod.containers['ipfs-cluster'].ports['proxy-http']` 給的 protocol 是你馬 UDP...(雖然能動...)
for res in `kubectl get po -Al app.kubernetes.io/name=ipfs-cluster-ipfs-cluster -o name`; do kubectl exec $res -n ipfs-operator-system -itc ipfs-cluster -- sed -i "s/\/ip4\/127.0.0.1\/tcp\/9095/\/ip4\/0.0.0.0\/tcp\/9095/g" /data/ipfs-cluster/service.json; done

kubectl rollout restart statefulset ipfs-cluster-ipfs-cluster -n ipfs-operator-system  # 因為沒有確認狀態，只會重啟最後一個QQ
kubectl delete po ipfs-cluster-ipfs-cluster-0 -n ipfs-operator-system # 需要砍掉第 0 個，以重啟並套用設定
```

- Check

```bash
wget http://ipfs-cluster-ipfs-cluster-0.ipfs-operator-system.svc.cluster.local:9095/api/v0/pin/ls
100.96.1.47:9095
```

## helm-ipfs-cluster - Setup

- 建立 Cluster Secret，並將它寫入 helm-ipfs-cluster/my-values.yaml 的 sharedSecret

```bash
od  -vN 32 -An -tx1 /dev/urandom | tr -d ' \n'
```

- 安裝

```bash
cd helm-ipfs-cluster
helm upgrade --install --create-namespace -n ipfs-cluster ipfs-cluster . -f my-values.yaml
```
