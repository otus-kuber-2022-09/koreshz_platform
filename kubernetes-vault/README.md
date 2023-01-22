# Hashicorp Vault & k8s
## Настройка Vault
Инициализация vault:
```bash
kubectl exec -it pod/vault-0 -- vault operator init --key-shares=1 --key-threshold=1
Unseal Key 1: /9jANF+O4/J/wCRZV89nxfFhKR3bFo0GJdQzMwPipKc=

Initial Root Token: hvs.MhNKOliYXV66kPTQ0ghbsDG1

Vault initialized with 1 key shares and a key threshold of 1. Please securely
distribute the key shares printed above. When the Vault is re-sealed,
restarted, or stopped, you must supply at least 1 of these keys to unseal it
before it can start servicing requests.

Vault does not store the generated root key. Without at least 1 keys to
reconstruct the root key, Vault will remain permanently sealed!

It is possible to generate new unseal keys, provided you have a quorum of
existing unseal keys shares. See "vault operator rekey" for more information.
```
Поды vault перейдут в состояние Ready только после того, как мы их распечатаем.
Распечатка vault:
```bash
kubectl exec -it pod/vault-0 -- vault operator unseal '/9jANF+O4/J/wCRZV89nxfFhKR3bFo0GJdQzMwPipKc='
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    1
Threshold       1
Version         1.12.1
Build Date      2022-10-27T12:32:05Z
Storage Type    file
Cluster Name    vault-cluster-2b43d467
Cluster ID      8e7af668-d008-1f5d-e578-0b9cfca1c644
HA Enabled      false
```
Аутентификация в vault:
```bash
kubectl exec -it pod/vault-0 -- vault login
Token (will be hidden):
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                hvs.MhNKOliYXV66kPTQ0ghbsDG1
token_accessor       TLgmtVhiU2kYPByUZ5krqAHb
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
```
Доступные виды аутентификации:
```bash 
kubectl exec -it pod/vault-0 -- vault auth list
Path      Type     Accessor               Description                Version
----      ----     --------               -----------                -------
token/    token    auth_token_55553700    token based credentials    n/a
```
Чтение секретов:
```bash
/ $ vault read otus/otus-ro/config
Key                 Value
---                 -----
refresh_interval    768h
password            asajkjkahs
username            otus
/ $ vault read otus/otus-rw/config
Key                 Value
---                 -----
refresh_interval    768h
password            asajkjkahs
username            otus
```
### Насторйка kubernetes авторизации в vault
Включение авторизации k8s:
```bash
/ $ vault auth enable kubernetes
Success! Enabled kubernetes auth method at: kubernetes/
/ $ vault auth list
Path           Type          Accessor                    Description                Version
----           ----          --------                    -----------                -------
kubernetes/    kubernetes    auth_kubernetes_30932d8f    n/a                        n/a
token/         token         auth_token_55553700         token based credentials    n/a
```
Запуск пода с правами serviceaccount:
```bash
kubectl run -ti --rm --image=alpine --overrides='{ "spec": { "serviceAccount": "vault-auth" }  }' tmp
```
Пытаемся залогиниться:
```bash
VAULT_ADDR=http://vault:8200
KUBE_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl --request POST  --data '{"jwt": "'$KUBE_TOKEN'", "role": "otus"}' $VAULT_ADDR/v1/auth/kubernetes/login | jq
```
В случае если все было правильно получам что-то вроде:
```json
{
  "request_id": "b81ff202-2647-be71-19d4-7b4b12431d0b",
  "lease_id": "",
  "renewable": false,
  "lease_duration": 0,
  "data": null,
  "wrap_info": null,
  "warnings": null,
  "auth": {
    "client_token": "hvs.CAESIPjr8cMUg5jr-KA-9NPwX2s0nYHXazL3qk_n4dnvSFwuGh4KHGh2cy41VDh6NFROOW5GaThyT3U4OVNJcFB5YTA",
    "accessor": "SbtdTiRzDIufVNrZ4mCF2WKC",
    "policies": [
      "default",
      "otus-policy"
    ],
    "token_policies": [
      "default",
      "otus-policy"
    ],
    "metadata": {
      "role": "otus",
      "service_account_name": "vault-auth",
      "service_account_namespace": "default",
      "service_account_secret_name": "",
      "service_account_uid": "370210ea-5e49-423c-a387-c63eff4aff45"
    },
    "lease_duration": 86400,
    "renewable": true,
    "entity_id": "ee6f0879-6481-50e8-b1e9-315155028907",
    "token_type": "service",
    "orphan": true,
    "mfa_requirement": null,
    "num_uses": 0
  }
}
```
## Получение vault секрета через kubernetes авторизацию
Логинимся и получаем клиентский токен:
```bash
TOKEN=$(curl -k -s --request POST --data '{"jwt": "'$KUBE_TOKEN'", "role": "otus"}' $VAULT_ADDR/v1/auth/kubernetes/login | jq '.auth.client_token'|awk -F\" '{print $2}')
```
Используя только что полученный токен попробуем прочитать секрет:
```bash
curl --header "X-Vault-Token:$TOKEN" $VAULT_ADDR/v1/otus/otus-ro/config | jq
```
```json
{
  "request_id": "55f565b2-32b2-d402-61cc-44b093c55188",
  "lease_id": "",
  "renewable": false,
  "lease_duration": 2764800,
  "data": {
    "password": "asajkjkahs",
    "username": "otus"
  },
  "wrap_info": null,
  "warnings": null,
  "auth": null
}
```
Теперь попробуем записать:
```bash
curl --request POST --data '{"bar": "baz"}' --header "X-Vault-Token:$TOKEN" $VAULT_ADDR/v1/otus/otus-ro/config
```
```json
{"errors":["1 error occurred:\n\t* permission denied\n\n"]}
```
```bash
curl --request POST --data '{"bar": "baz"}' --header "X-Vault-Token:$TOKEN" $VAULT_ADDR/v1/otus/otus-rw/config
```
```json
{"errors":["1 error occurred:\n\t* permission denied\n\n"]}
```
```bash
curl --request POST --data '{"bar": "baz"}' --header "X-Vault-Token:$TOKEN" $VAULT_ADDR/v1/otus/otus-rw/config1
```
По понятным причинам мы не смогли написать в `outus/ro`, при этом в мы смогли записать в `otus/otus-rw/config1` потому, что в соответсвии с `otus-policy` в `otus/otus-rw/*` мы можем создавать. Для того, чтобы успешно обновить значение `otus/otus-rw/config` необходимо в политике к данному пути добавить `update`:
```json
path "otus/otus-ro/*" {
  capabilities = ["read", "list"]
}
path "otus/otus-rw/*" {
  capabilities = ["read", "create", "list", "update"]
}
```
###  Авторизация через kuber-agent:
Качаем примеры:
```bash
git clone https://github.com/hashicorp/vault-guides.git
cd vault-guides/identity/vault-agent-k8s-demo
```
Изменяем конфиги:
```diff
diff --git a/identity/vault-agent-k8s-demo/configmap.yaml b/identity/vault-agent-k8s-demo/configmap.yaml
index 52cf824..7ffb8b4 100644
--- a/identity/vault-agent-k8s-demo/configmap.yaml
+++ b/identity/vault-agent-k8s-demo/configmap.yaml
@@ -10,7 +10,7 @@ data:
         method "kubernetes" {
             mount_path = "auth/kubernetes"
             config = {
-                role = "example"
+                role = "otus"
             }
         }

@@ -27,10 +27,10 @@ data:
     <html>
     <body>
     <p>Some secrets:</p>
-    {{- with secret "secret/data/myapp/config" }}
+    {{- with secret "otus/otus-ro/config" }}
     <ul>
-    <li><pre>username: {{ .Data.data.username }}</pre></li>
-    <li><pre>password: {{ .Data.data.password }}</pre></li>
+    <li><pre>username: {{ .Data.username }}</pre></li>
+    <li><pre>password: {{ .Data.password }}</pre></li>
     </ul>
     {{ end }}
     </body>
diff --git a/identity/vault-agent-k8s-demo/example-k8s-spec.yaml b/identity/vault-agent-k8s-demo/example-k8s-spec.yaml
index f99fa50..25fc797 100644
--- a/identity/vault-agent-k8s-demo/example-k8s-spec.yaml
+++ b/identity/vault-agent-k8s-demo/example-k8s-spec.yaml
@@ -23,7 +23,7 @@ spec:
     - -log-level=debug
     env:
     - name: VAULT_ADDR
-      value: http://EXTERNAL_VAULT_ADDR:8200
+      value: http://vault:8200
     image: vault
     name: vault-agent
     volumeMounts:
```
Создаем configmap:
```bash
kubectl apply -f configmap.yaml
```
Запускаем valt-example-pod:
```bash
kubectl apply -f example-k8s-spec.yml
```
Инит контейнер получает секреты из vault и формирует страницу, содежащую их. Для проверки получаем сгенерированную страницу:
```bash
kubectl exec -it pod/vault-agent-example -- cat /usr/share/nginx/html/index.html
```
```html
<html>
<body>
<p>Some secrets:</p>
<ul>
<li><pre>username: otus</pre></li>
<li><pre>password: asajkjkahs</pre></li>
</ul>

</body>
</html>
```
## Создаем CA на основе vault
### Создаем корневой сертификат
Включаем pki авторизацию:
```bash
vault secrets enable pki
```
Немножко поднастроим, чтобы выдавать сертификаты с максимальным временем жизни:
```bash
vault secrets tune -max-lease-ttl=87600h pki
```
Генерируем корневой сертификат:
```bash
kubectl exec -it vault-0 -- vault write pki/root/generate/internal common_name="example.com" ttl=87600h > CA_cert.crt
```
Просмотреть информацию об имеющихся корневых сертификатах:
```bash
vault list pki/issuers
Keys
----
148ca271-01f6-924b-87a0-af7590c2ae72
```
Прописываем ссылки на нашего ca и отозванных сертификатов:
```bash
vault write pki/config/urls issuing_certificates="http://vault:8200/v1/pki/ca" crl_distribution_points="http://vault:8200/v1/pki/crl"
Success! Data written to: pki/config/urls
```
```bash
vault read pki/config/urls
Key                        Value
---                        -----
crl_distribution_points    [http://vault:8200/v1/pki/crl]
issuing_certificates       [http://vault:8200/v1/pki/ca]
ocsp_servers               []
```
### Создаем промежуточный сертификат
Действия аналогичны получения ca, однако на последнем шаге получаем Certificate Signing Request:
```bash
$ vault secrets enable --path=pki_int pki
Success! Enabled the pki secrets engine at: pki_int/
```
```bash
$ vault secrets tune -max-lease-ttl=87600h pki_int
Success! Tuned the secrets engine at: pki_int/
```
```bash
$ kubectl exec -it vault-0 -- vault write -format=json pki_int/intermediate/generate/internal common_name="example.com Intermediate Authority"         | jq -r '.data.csr' > pki_intermediate.csr
```
Подписываем полученный csr нашим ca:
```bash
$ kubectl cp pki_intermediate.csr vault-0:./
$ kubectl exec -it vault-0 -- vault write -format=json pki/root/sign-intermediate csr=@pki_intermediate.csr format=pem_bundle ttl="43800h" |  jq -r '.data.certificate' > intermediate.cert.pem
$ kubectl cp intermediate.cert.pem vault-0:./
$ kubectl exec -it vault-0 -- vault write pki_int/intermediate/set-signed certificate=@intermediate.cert.pem
```
```
Key                 Value
---                 -----
imported_issuers    [f659ea39-f008-c830-8070-e34beb02536d 2604c58d-2be6-67ec-0990-ecba43cc116f]
imported_keys       <nil>
mapping             map[2604c58d-2be6-67ec-0990-ecba43cc116f: f659ea39-f008-c830-8070-e34beb02536d:c6c204a5-7590-9097-3495-a9b728b510e2]
```
### Подписываем и отзываем сертификаты
Создадим роль для выдачи сертификатов:
```bash
$ vault write pki_int/roles/example-dot-com allowed_domains="example.com" allow_subdomains=true max_ttl="720h"
```
Создадим сертификат на имя `gitlab.devlab.ru`:
```bash
$ vault write pki_int/issue/example-dot-com common_name="gitlab.example.com" ttl="24h"
```