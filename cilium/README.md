# cilium

## 說明

- 由於目前的網路是交由 kops 去維護 cilium cni 的，所以這裡不再去安裝 cni
- 僅使用 cilium 的 hubble，以提供介面去觀測網路
- 原則上版本要跟 kops 所提供的 cilium 版本一樣，以避免問題

## Install

- 使用以下指令

```bash
helm upgrade --install --namespace kube-system --repo https://helm.cilium.io cilium cilium --version 1.12.1 -f my-values.yaml
# helm upgrade --install --namespace kube-system --repo https://helm.cilium.io cilium cilium --version 1.14.0 -f my-values.yaml
```
