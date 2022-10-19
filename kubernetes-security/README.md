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
- RBAC - 
- Webhook