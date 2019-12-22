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
