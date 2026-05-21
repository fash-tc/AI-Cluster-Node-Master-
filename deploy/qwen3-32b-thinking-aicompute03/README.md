# Qwen3-32B (hybrid thinking) on aicompute03

Fast tier reasoning replica B. Pair of `qwen3-32b-thinking-aicompute02`.

## Endpoint

- Direct: `http://aicompute03.cnco1.tucows.cloud:31443/v1`
- Through gateway: same as 02, gateway round-robins.

## Apply

```powershell
& 'C:\Users\fash\Desktop\AI Nodes\tools\kubectl.exe' --kubeconfig 'C:\Users\fash\Desktop\AI Nodes\kubeconfig-prod-ai-k8-windows' apply -k 'C:\Users\fash\Desktop\AI Nodes\deploy\qwen3-32b-thinking-aicompute03'
```

## Scale up

```powershell
& 'C:\Users\fash\Desktop\AI Nodes\tools\kubectl.exe' --kubeconfig 'C:\Users\fash\Desktop\AI Nodes\kubeconfig-prod-ai-k8-windows' -n lab-domains-sre scale deploy/qwen3-32b-thinking-aicompute03 --replicas=1
```
