# Deployment 
Если deployment запускается и не проходит liveness или rediness проверки, то при обновлении deployment не удаляются replicaset-ы
# Kubernetes networks
Для получения сетевого доступа к поду необходимо использовать Service.
## Типы Service
### ClusterIP
Используется для взаимодействия подов внутри кластера. Адрес выделяется из внутреннего пула кластера. За одним сервисом может находиться несколько подов. Поды выбираются сервисом через селектор. Запросы к подам распределяются случайным образом. 
ClusterIP адресов на самом деле не существует. Адреса транслируются из clusterip в настоящие адреса нод при помощи iptables на ноде, с которой идет запрос в цепочках KUBE-SEP-...
### NodePort
Предоставляет доступ к службе через адреса нод на статически заданном порту NodePort
### LoadBalancer
Предоставляет доступ к службе через облачный балансировщик нагрузки (или metallb)
### ExternalName
Используется в случае необходимости обращения из кластера к внешнему ресурсу
## Headless Service
Служба без clusterip, используется "интеллектуально" балансировкой, например ingress
```
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  selector:
    app: web
  type: ClusterIP
  clusterIP: None
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
```
## Ingress
Объект предоставляющий доступ к кластеру извне, обычно через http или https
```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: web
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /web
        backend:
          serviceName: web-svc
          servicePort: 8000
```
### Sticky session
Для того, чтобы вэб сессия клиента всегда работала с одним и тем же подом неоходимо реализовывать sticky-sessions через cookies. Для этого в определение ingerss необходимо добавить следующие annotations:
```
  annotations:
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-name: "route"
    nginx.ingress.kubernetes.io/session-cookie-max-age: "172800"
```
