Aplicação de um manifesto

```bash
    kubectl apply -f dashboard-adminuser.yaml
```


```bash
    kubectl apply -f admin-user-token-secret.yaml
```

Para acessar o Dashboard, você precisará de um token da ServiceAccount que acabamos de criar.


```bash
    # Primeiro, encontre o nome do secret associado à ServiceAccount
kubectl -n kubernetes-dashboard get secret $(kubectl -n kubernetes-dashboard get sa admin-user -o jsonpath="{.secrets[0].name}") -o jsonpath="{.data.token}" | base64 --decode
```
