# traefik

## 說明

- 就是個 Ingress
- 需要注意因為 external-dns 會需要得到 ingress 的 svc 狀態，因此需要開啟 publishedService 的功能，以讓 external-dns 取得資料並去改 dns 紀錄

## Setup

- 使用以下指令配置 traefik ingress-controller

```bash
helm upgrade --install --create-namespace --namespace traefik --repo https://traefik.github.io/charts traefik traefik --version 24.0.0 -f my-values.yaml
```
