# Volumes, Storages, StatefulSet
## PersistentVolume and PersistentVolumeClaim
- PersistentVolume - часть хранилища, выделенная администратором или динамически в соотвествии с Storage Classes. Их жизненный цикл независим от жизненного цикла подов.
- PersistentVolumeClaim - запрос пользователя на хранилице. Аналогичны подам - поды потребляют ресурсы, а PVC потребляют ресурсы PV. Могут быть смонтированны в нескольких режимах:
  - ReadWriteOnce
  - ReadOnlyMany
  - ReadWriteMany (поддерживается только продвинутыми хранилищами, например [CephFS](https://habr.com/ru/post/179823/))
[StorageClass](https://kubernetes.io/docs/concepts/storage/storage-classes/) ресурсы используются для предоставления различных видов хранилищ (например медленное и быстрое, расположенное в различных датацентрах)
## Жизненный цикл PersistentVolumeClaim и PersistentVolume
- Выделение
  - Статическое (администратором)
  - Динамическое (в соотвествии с StorageClass)
- Связывание (binding) pvc с pv. Систем ищет соответсвующий запросу pv и резервирует его эксклюзивно (даже если pv зарезервировал больше хранилица, чем необходимо) в соотвествии с выбранным режимом доступа.
- Использование. Хранилище монтируется в под в виде тома. 
- Освобождение (reclaiming) по окончании работы под может удалить pvc. Reclaim policy сообщает системе, что необходимо сделать с хранилищев после удаления pvc:
  - `Retain` - после окончания работы том все еще существует, однако не может быть использован повторно пока администратор его не удалит или очистит.
  - `Delete` - том автоматически удаляется вместе со всеми данными
  - `Recycle` - том автоматически очищается и ожидает новый запрос. 

## Использование секретов
Для использования секретов в поде необходимо их предварительно создать, например команда `kubectl create secret generic minio-credentials --from-literal=accesskey=minio --from-literal=secretkey=minio123` создает секрет из литералов

После этого секрет можно использовать в манифесте:
- В виде смонтированного файла:
    ```apiVersion: v1
    kind: Pod
    metadata:
    name: mypod
    spec:
    containers:
    - name: mypod
        image: redis
        volumeMounts:
        - name: foo
        mountPath: "/etc/foo"
        readOnly: true
    volumes:
    - name: foo
        secret:
        secretName: mysecret
        optional: false # default setting; "mysecret" must exist
    ```
- в виде переменных среды окружения:
    ```
    apiVersion: v1
    kind: Pod
    metadata:
    name: secret-env-pod
    spec:
    containers:
    - name: mycontainer
        image: redis
        env:
        - name: SECRET_USERNAME
            valueFrom:
            secretKeyRef:
                name: mysecret
                key: username
                optional: false # same as default; "mysecret" must exist
                                # and include a key named "username"
        - name: SECRET_PASSWORD
            valueFrom:
            secretKeyRef:
                name: mysecret
                key: password
                optional: false # same as default; "mysecret" must exist
                                # and include a key named "password"
    restartPolicy: Never
    ```

