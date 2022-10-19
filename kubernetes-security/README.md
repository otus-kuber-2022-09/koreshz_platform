# Kubernetes security
Три стадии авторизации:
1. authentication 
2. authorization 
3. admission control
## Аутентификация
Аутентификация происходит через протокол https при помощи одного или нескольких плагинов
## Авторизация
Виды авторизации:
- [Node](https://kubernetes.io/docs/reference/access-authn-authz/node/) 
- Attribute-based access control (ABAC) - права доступа выдаются через политики, политики связывают права и набор атрибутов
- RBAC
- Webhook

# Работа с rbac
Для получения существующих ролей использовать комманду:
`kubectl get clusterrole`
Имеются уже готовые роли admin и view
Для проверки имеет ли пользователь права на действие использовать команду:
`kubectl auth can-i create pods --as system:serviceaccount:dev:ken -n dev` => может ли сервисный аккаунт кен из пространства имен дев создавать поды
Для применения роли ко всем пользователям пространства имен prometheus:
```
- kind: Group
  name: system:serviceaccounts:prometheus
  apiGroup: rbac.authorization.k8s.io
```
