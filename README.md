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
