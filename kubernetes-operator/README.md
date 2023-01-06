# kubernetes-operator
## Custom resource definition
Custom resources является способом расширения api kubernetes. Являются restful объектами kubernetes api.
Действие ресурса может быть ограничего одним пространстом имен (`Namespaced`) или распространяться на весь кластер (`Cluster`), как указывается в `spec.scope`.
Описание пользовательского ресурса производится в `spec.versions.schema.openAPIV3Schema`. Здесь мы описываем пользовательские поля нашего ресуса: их имена, типы, а также являются ли они обязательными для создания ресурса `required`
Подробное описание crd можно посмотреть [здесь](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)
### Манифест crd из задания с комментариями:
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: mysqls.otus.homework
spec:
  # either Namespaced or Cluster
  scope: Namespaced
  # group name to use for REST API: /apis/<group>/<version>
  group: otus.homework
  versions:
    - name: v1
      # Each version can be enabled/disabled by Served flag.
      served: true
      # One and only one version must be marked as the storage version.
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            apiVersion:
              type: string
            kind:
              type: string
            metadata:
              type: object
              properties:
                name:
                  type: string
            spec:
              type: object
              properties:
                image:
                  type: string
                database:
                  type: string
                password:
                  type: string
                storage_size:
                  type: string
              required:
                - database
                - password
                - storage_size
  names:
    # kind is normally the CamelCased singular type. Your resource manifests use this.
    kind: MySQL
    # plural name to be used in the URL: /apis/<group>/<version>/<plural>
    plural: mysqls
    # singular name to be used as an alias on the CLI and for display
    singular: mysql
    # shortNames allow shorter string to match your resource on the CLI
    shortNames: 
     - ms
```
### Манифест описания созданного crd:
```yaml
apiVersion: otus.homework/v1
kind: MySQL
metadata:
  name: mysql-instance
spec:
  image: mysql:latest
  database: otus-database
  password: 0tusP@ssword
  storage_size: 1Gi
```
## Custom controller
CustomResouceDefinition является структурированным статическим деревом данных. Логику работы crd задает Custom controller - программа создающая, управляющая ресурсом. Например при создании crd контроллер создаст pv и pvc, а при удалении удалит их.
Для создания контроллера можно использовать один из существующих фреймворков.
При выполненнии задания использовался kopf (kubernetes operator framework)
Для управления ресурсами (создания и управления объектами kubernetes) использовалась библиотека kubernetes-client.
Для обработки событий kops использует декораторы: 
- `@kopf.on.create('otus.homework', 'v1', 'mysql')` - для обработки создания crd
- `@kopf.on.delte('otus.homework', 'v1', 'mysql')` - для обработки удалния ресурса

Для того, чтобы не следить за удалением вспомогательных ресурсов можно использовать систему влдений.
`kopf.append_owner_reference(persisten_volume, owner=body)` назначает body владельцем persistent_volume, и при удалении body автоматически удалится persistent_volume
## Задание
В результате выполнения задания был создан mysql-operator - набор crd и контроллер, создающий необходимые для работы mysql сервера service, deployment, pv и pvc. Кроме того контроллер восстанавливает базу данных из резервной копии при создании mysql crd и сохраняет резервную копию при удалении ресурса. 