kubectl edit application ingress-nginx -n argocd => xóa ingerss

argo pass: 
```
╰─ kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d && echo                                                                 ─╯

ibhLYtMlTn7mdpag

hGuCynWIdoNCXmuO (pass argo cty)

argocd login localhost:8080 --username admin --password ibhLYtMlTn7mdpag  
```


argo 8080
thread-fe 8081
drone 8082

#### Harbor
project Threads, secret robot account
lFdeqWCaMmYLtw3IynhGiixZYX8yf7mR