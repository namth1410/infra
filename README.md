kubectl edit application ingress-nginx -n argocd => xóa ingerss

argo pass: 
```
╰─ kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d && echo                                                                 ─╯

ibhLYtMlTn7mdpag
```