# kubernetes templating
## Работа с helm
Добавить репозиторий:
```
helm repo add [repository-name] [url]
```

Удалить пепозиторий из системы:
```
helm repo remove [repository-name]
```

Обновить репозитории:
```
helm repo update
```

Установка чарта:
```
helm install [app-name] [chart]
```

Установка чарта в выбранный namespace:
```
helm install [app-name] [chart] --namespace [namespace]
```

Переопределить значения по умолчанию значениями из файла:
```
helm install [app-name] [chart] --values [yaml-file/url
```

"Холостой" запуск (выводит полученный yaml в stdout не применяя его):
```
helm install [app-name] --dry-run --debug
```

Удаление чарта из выбранного namespace:
```
helm uninstall [release] --namespace [namespace]
```
## Работа с cert-manager
Для того, чтобы cert-manager выдавал нам сертификаты необходимо создать Issuer или даже ClusterIssuer, если мы хотим выдавать сертификаты на весь кластер.
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    # You must replace this email address with your own.
    # Let's Encrypt will use this to contact you about expiring
    # certificates, and issues related to your account.
    email: mail@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      # Secret resource that will be used to store the account's private key.
      name: example-issuer-account-key
    # Add a single challenge solver, HTTP01 using nginx
    solvers:
    - http01:
        ingress:
          class: nginx
```
Во время отладки рекомендуется использовать [staging](https://letsencrypt.org/docs/staging-environment/) окружение (тестовое), чтобы letsencrypt нас не забанил 
## Установка chartmuseum и настройка ingress через helm для автоматического получения сертификатов
Для автоматического получения сертификатов (в нашем случае от letsencrypt) в аннотации ingress публикуемого сервиса необходимо добавить следующие значения:
```yaml
ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: nginx
    kubernetes.io/tls-acme: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-production"
    cert-manager.io/acme-challenge-type: http01

  hosts:
    - name: chartmuseum.62.84.120.150.nip.io
      path: /
      tls: false
    - name: chartmuseum.62.84.120.150.nip.io
      path: /

      ## Set this to true in order to enable TLS on the ingress record
      tls: true

      ## If TLS is set to true, you must declare what secret will store the key/certificate for TLS
      ## Secrets must be added manually to the namespace
      tlsSecret: chartmuseum.62.84.120.150.nip.io
```
Необходимо использовать данные значения при установки chartmuseum через helm (передавать value.yaml).
При создании ингресса cert-manager автоматически запросит сертификат и сохранит его в tlsSecret. В дальнейшем закрытый ключ и сертификат будут использоваться для установления безопасного tls соединения.
Значение `aсme-challenge-type: http01` в аннотациях задает способ [валидации dns имени](https://letsencrypt.org/docs/challenge-types/) сервер letsencrypt выдает некий токен, который публикуется по http на нашем dns, и таким образом удостоверяется, что данное имя принадлежит нам.
## Создание собственного helm chart
Для создания helm chart используем команду:
```
helm create kubernetes-templating/hipster-shop
```
При написании шаблонов используем плэйсхолдеры типа `{{ .Values.service.type }}` которые будут наполняться из `values.yaml`. Где `.Values` - корень файла `values.yaml`
Получаем шаблон следующиего вида:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  labels:
    app: frontend
spec:
  type: {{ .Values.service.type }}
  selector:
    app: frontend
  ports:
  - name: http
    port: {{ .Values.service.port }}
    targetPort: {{ .Values.service.targetPort }}
    {{ if eq .Values.service.type "NodePort" }}
    nodePort: {{ .Values.service.nodePort }}
    {{ end }}
```
Который будет заполняться из `values.yaml` файла:
```yaml
service:
  type: ClusterIP
  port: 80
  targetPort: 8080
```
## Зависимости
В helm 3 зависимости хранятся в файле `Chart.yaml`:
```yaml
apiVersion: v2
name: hipster-shop
description: A Helm chart for Kubernetes
type: application
version: 0.1.0
appVersion: "1.16.0"

dependencies:
  - name: frontend
    version: 0.1.0
    repository: "file://../frontend"
```
В данном случае зависимость указывает на локальную директорию `file://../frontend`
Перед установкой чарта с зависимостями необходимо его обновить
```
helm dep update kubernetes-templating/hipster-shop
```
В результате обновления чарт скачает (если надо) и соберет все зависимости в директорию `charts` в виде tgz архивов.
## kustomize
Инструмент, позволяющий шаблонизировать манифесты, оставляя оригинальные файлы без изменений. 
Позволяет создать несколько версия одной и тойже базы. Например для использования одних и тех же файлов в различных средах исполнения dev и production. Для этого необходимо создать в проекте следующую структуру каталогов:
```
project_name/
├── base/
├── overlays/
│   ├── dev/
│   ├── prod/
```
В директорию `base/` помимо манифестов проекта небоходимо поместить файл `kustomization.yaml`, содержащий перечень манифестов, подлежащих шаблонизации:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - recomendationservice-svc.yaml
  - recomendationservice-depl.yaml
```
Создать директорию (или несколько) содержащую специфичные для данной среды исполнения манифесты (например создающие пространство имен dev), а также файлы шаблонизации (изменяющие тег оригинального base образа, добавляющий лейблы, добавляющие префиксы ресурсам):
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: gitops-dev
bases:
  - ../../base

resources:
  - namespace.yaml
images:
  - name: gcr.io/google-samples/microservices-demo/recommendationservice
    newTag: gitops-dev
commonLabels:
  env: dev
namePrefix: dev-
```
значение `bases` указывает на директорию с исходными (шаблонизируемыми) манифестами;
значение `resources` указывает какие манифесты должны быть применены в ходе шаблонизации (в данном случае создастся namespace)
значение `images` изменяет тег образа на значение из `newTag`
`commonLables` задает лейблы для всех кастомизированных ресурсов
`namePrefix` задает префикс имени всех кастомизированных ресурсов
Для получения кастомизированного манифеста необходимо находясь в директории проекта воспользоваться коммандой `kubectl kustomize overrides/dev`
Для применения кастомизированного манифеста необходимо воспользоваться командой `kubectl apply -k overrides/dev`