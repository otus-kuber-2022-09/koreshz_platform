# Prometheus
Установка prometheus-operator через helm:
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install [RELEASE_NAME] prometheus-community/kube-prometheus-stack
```
## Подготовка nginx
Конфиг в nginx будем пробрасывать через ConfigMap а не через кастомный образ докера (хотя такое решение тоже готово, см. nginx-stub-status/Dockerfile)
```
apiVersion: v1 
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    server { 
      listen 80;
      server_name frontend;
      location = / {
      root /usr/share/nginx/html;
      try_files $uri /index.html;
      }
        location = /basic_status {
        stub_status;
      }
    }
```
Пробрасываем конфиг в поды деплоймента, метрики с nginx-а будем собирать при помощи nginx-exporter установленного в sidecar в каждом поде. Т.к. оба образа (nginx и exporter) запущены в одном поде и имеют единое сетевое пространство задаем -nginx.scrape-uri=http://localhost/basic_status :
```
apiVersion: apps/v1
kind: Deployment
metadata:
    name: nginx-deployment
    labels:
      app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
      annotations:
        prometheus.io/scrape: 'true'
        prometheus.io/port: '9113'
    spec:
      containers:
        - name: nginx-stub-status
          image: nginx 
          ports:
            - containerPort: 80
          volumeMounts:
            - name: nginx-config
              mountPath: /etc/nginx/conf.d/default.conf
              subPath: nginx.conf
        - name: nginx-exporter
          image: nginx/nginx-prometheus-exporter:0.11
          args:
            - '-nginx.scrape-uri=http://localhost/basic_status'
          ports:
            - containerPort: 9113
      volumes:
        - configMap: 
            defaultMode: 420
            name: nginx-config
          name: nginx-config
```
При создании service обязательно необходимо дать имя порту на котором будут доступны метрики:
```
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  labels:
    app: nginx
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    name: http
  - port: 9113
    targetPort: 9113
    name: metrics
```
## Prometheus
Мониторинг объектов в prometheus осуществляется по лэйблам. Существующий после установки через helm прометей будет мониторить приложение, если его лейблы находятся в `serviceMonitorSelelector.matchLables`:
```
...
kind: Prometheus
...
    serviceMonitorSelector:
      matchLabels:
        release: prometheus
...
```


Создадим ServiceMonitor, который будет отслеживать состояние наших подов (через экспортер) по лейблам (создаваемый хелмом прометей мониторит только лэйбл `release=prometheus` ). В `spec.selector.matchLabels` должны быть лейблы нашей службы и подов. В нашем случае `app: nginx`
Не забываем, что в значение `port` в `endpoints` должно быть указано через имя порта указанного в службе, т.е. в данном случае `metrics`:
```
    endpoints:
      - port: metrics
        path: /metrics
        scheme: http
```
Создаем ServiceAccount, ClusterRole и ClusterRoleBinding. Для обеспечения необходимых прав доступа. 
Теперь собственно сам объект prometheus:
```
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
spec:
  serviceAccountName: prometheus
  serviceMonitorSelector:
    matchLabels:
      app: nginx
  resources:
    requests:
      memory: 400Mi
  enableAdminAPI: false
```
Есть неплохая [статья](https://github.com/prometheus-operator/prometheus-operator/blob/main/Documentation/troubleshooting.md#troubleshooting-servicemonitor-changes) по траблшутингу