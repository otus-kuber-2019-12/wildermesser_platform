# wildermesser_platform
wildermesser Platform repository

## Поды в kubernetes
В директории `kubernetes-intro` находятся манифесты для:
 - создания пода с примитивным http сервером. Содержимое генерируется в `init` контейнере
 - создания пода с frontend частью проекта Hipster-Shop

## Конроллеры в kubernetes
В директории `kubernetes-controllers` находятся манифесты для:
 - создания ReplicaSet для `frontend` и `paymentservice`
 - создания Deployment для `paymentservice` с различными стратегиями деплоя
 - создания Daemonset для `node-exporter` запускающего поды в том числе на `control-plane` нодах
## RBAC
В директории `kubernetes-security` находятся манифесты для различных политик управления доступом к k8s api
## Networks
В директории `kubernetes-networks` находятся манифесты для различных типов сервисов:
 - ClusterIP (обычный и headless)
 - LoadBalancer
Также манифест для Ingress.
В директории `kubernetes-networks/coredns` манифесты серисов типа LoadBalancer для публикации CoreDNS на одном и том же
адресе балансировщика Metallb
В директории `kubernetes-networks/dashboard` манифесты для публикации kubernetes-dashboard через Ingress
В директории `kubernetes-networks/canary` манифесты для Canary deployment с помощью Ingress
## Volumes
В директории `kubernetes-volumes` находятся манифесты для деплоя `minio` как `statefulset`, причём данные для авторизации
сохранены в ресурсе `secret` 
## Templating
В директории `kubernetes-templates` находятся файлы для шаблонизации манифестов различными способами:
 - helm
 - jsonnet
 - kustomize
## Operators
В директории `kubernetes-operators` находится реализация простого MySQL оператора, который обрабатывае события создания
и удаления CRD mysqls.otus.homework.
При создании ресурсе проиходит попытка восстановления из бэкапа. При удалении - создание бэкапа/
Пример:
```
NAME                         COMPLETIONS   DURATION   AGE
backup-mysql-instance-job    1/1           1s         16m
restore-mysql-instance-job   1/1           44s        72s
```
После создания CR с теми же параметрами проиходит восстановление из бэкапа и в таблице есть данные:
```
+----+-------------+
| id | name        |
+----+-------------+
|  1 | some data   |
|  2 | some data-2 |
+----+-------------+
```
В случае успешного создания, в статусе CR появляется запись вида:
```
Status:                                                                                                                                                    │
  Kopf:                                                                                                                                                    │
    Dummy:  2020-02-10T07:55:01.509237                                                                                                                     │
  mysql_on_create:                                                                                                                                         │
    Message:   mysql-instance created without restore-job    
```
Это реализуется возвращением словаря в функции-обработчике mysql_on_update

В случае, если меняется пароль к базе данных, то:
1. С помощью `list_namespaced_pod('default',label_selector='app=%s' % name)` находится нужный под с mysql
2. С помощью `kubernetes.stream` в этоп поде запускается команда для смены пароля `mysqladmin -u root -p%s password %s' % (old_password, password)`
3. Обновляется деплоймент для синхронизации переменных окружения (в данном случае MYSQL_ROOT_PASSWORD применятеся только при первом старте, но в общем случае логично обновлять весь деплоймент)
4. С помощью `kubernetes.client.V1Scale` осуществляется скалирование пода в 0, а потом в 1 для применения новых значений переменных окружения
5. В `describe mysql` добавляется статус:
```
  mysql_on_update:                                                                                                                                         │
    Message:  password updated
```
