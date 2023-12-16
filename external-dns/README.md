# external-dns

## 說明

- 目前會將 IngressHost 接到 Cloudfare 上，以達到自動化同步

## Setup

- 先[參考說明](https://kubernetes-sigs.github.io/external-dns/v0.13.6/tutorials/cloudflare/#creating-a-cloudflare-dns-zone)建立 Cloudflare Credentials

```bash
export API_TOKEN=xxxxx
kubectl create ns external-dns
kubectl create secret generic external-dns-env --from-literal="CF_API_TOKEN=$API_TOKEN" -n external-dns # -n cert-manager
```

- 使用以下指令配置 cloudfare

```bash
helm upgrade --install --create-namespace --namespace external-dns --repo https://kubernetes-sigs.github.io/external-dns external-dns external-dns --version 1.13.1 -f my-values.yaml
```
