# Сервисы централизованного логирования для Kubernetes

> В домашнем задании развернем сервисы для централизованного логирования (EFK/Loki) внутри Kubernetes, научимся отправлять туда структурированные логи продукта и инфраструктурных компонентов, рассмотрим способы визуализировать информацию из логов.

Создадим кластер `otus-kuber` в Yandex.Cloud. В кластер добавим группу узлов `default-pool` из одного узла и группу `infra-pool` из трех узлов. В последней группе сразу при создании добавим taint-политику узла `node-role=infra:NO_SCHEDULE`.

[Установим](https://cloud.yandex.ru/docs/cli/quickstart#install) `yc` и [настроим](https://cloud.yandex.ru/docs/managed-kubernetes/operations/connect/#kubectl-connect) подключение к кластеру с помощью `kubectl`. Убедимся, что всё работает:

```
$ kubectl get nodes -A
NAME                        STATUS   ROLES    AGE     VERSION
cl16uhlqmer9jepl0gg0-akeh   Ready    <none>   7m      v1.21.5
cl16uhlqmer9jepl0gg0-inix   Ready    <none>   6m39s   v1.21.5
cl16uhlqmer9jepl0gg0-yhig   Ready    <none>   7m21s   v1.21.5
cl1ggc8gf4a1omnioek6-uliv   Ready    <none>   10m     v1.21.5
```

Деплоим тестовое приложение в новый NS:

```
$ kubectl create ns microservices-demo
$ kubectl apply -f https://raw.githubusercontent.com/express42/otus-platform-snippets/master/Module-02/Logging/microservices-demo-without-resources.yaml -n microservices-demo
```

Для сервиса `adservice` не нашлось образа (`ErrImagePull`). Сервис `loadgenerator` упал с ошибкой оболочки `[[: not found`. Делаем вид, что всё хорошо, и идём дальше.

~~Притворяемся финами и~~ устанавливаем EFK-стек:

```
$ helm repo add elastic https://helm.elastic.co
$ helm repo add stable https://charts.helm.sh/stable
$ helm repo update
$ kubectl create ns observability
$ helm upgrade --install elasticsearch elastic/elasticsearch --namespace observability
$ helm upgrade --install kibana elastic/kibana --namespace observability
$ helm upgrade --install fluent-bit stable/fluent-bit --namespace observability
```

Ожидаемо обламываемся:

> Warning  FailedScheduling   49s (x4 over 2m57s)  default-scheduler   0/4 nodes are available: 1 Insufficient memory, 3 node(s) had taint {node-role: infra}, that the pod didn't tolerate.

И пытаемся исправить ситуацию с помощью [values](./elasticsearch.values.yaml), где указаны `tolerations` для "отравленных" нод из группы `infra-pool` (для селектора нод оказалось возможным использовать не понятное нам имя группы, а ее уканильный идентификатор, который можно посмотреть в свойствах группы узлов):

```
$ helm upgrade --install elasticsearch elastic/elasticsearch --namespace observability -f ./elasticsearch.values.yaml
$ helm upgrade --install kibana elastic/kibana --namespace observability -f ./kibana.values.yaml
$ helm upgrade --install fluent-bit stable/fluent-bit --namespace observability -f ./fluent-bit.values.yaml
```

Показалось, что statefulset от elasticsearch и daemonset от fluent-bit не меняют своего состояния. Переустанавливаем чарты, предварительно удалив релизы.

Заодно понимаем, что зря пожадничали, выделив на ноды в этой группе по 2ГБ оперативки, и увеличиваем до 8 ГБ. Ждём, пока изменения применятся, и в кластере наступит тишина:

```
$ kubectl get pods -n observability -o wide -l chart=elasticsearch -w
```

(Благодарность `@Листраткин Евгений` за идею с подменой `image` в values-файле для решения проблемы 403 при попытке спуллить образы эластика из инфры Яндекса)

Теперь деплоим ingress-контроллер:

```shell
$ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
$ helm repo update
$ helm install ingress-nginx ingress-nginx/ingress-nginx -f ./nginx.values.yaml
```

Чтобы ingress заработал, нужно [дать права](https://cloud.yandex.ru/docs/iam/operations/roles/grant) администрирования [Network Load Balancer](https://cloud.yandex.ru/docs/network-load-balancer/security/) сервисному пользователю. Для этого открываем нашу "папку", выбираем меню "Права доступа", жмём "Назначить роли", выбираем пользователя `k8s-cluster-*` и роль `load-balancer.admin`. Нажимем "Сохранить" и возвращаемся в консоль кластера (раздел "Сеть"), чтобы убедиться, что сервис "ingress-nginx-controller" стал зеленым. После этого помещаем его публичный адрес в [values](./kibana.values.yaml) и обновляем релиз кибаны:

```
$ helm upgrade --install kibana elastic/kibana --namespace observability -f ./kibana.values.yaml
```

Встречаем наш дашборд кибаны: http://kibana.51.250.85.229.nip.io/ (HTTPS настраивать не будем, потому что).

Задание со ⭐ пропускаем...

Деплоим `prometheus-operator`:

```
$ helm upgrade --install prometheus-operator stable/prometheus-operator -f ./prometheus-operator.values.yaml
```

И `elasticsearch-exporter`:

```
$ helm upgrade --install elasticsearch-exporter --namespace=observability stable/elasticsearch-exporter -f ./elasticsearch-exporter.values.yaml
```

Теперь нам нужно импортировать дашборд в графану. Но когда мы ее деплоили, она у нас взялась? Не спрашивайте. Добавляем ingress в [values](./prometheus-operator.values.yaml) (без `ingressClassName` не заработает) и обновляем релиз. Теперь у нас есть графана, вот она: http://grafana.51.250.85.229.nip.io/ (`admin:prom-operator`)

Дашборд импортируем по ID `4358`. На дашборде во всех виджетах - "No data" (с устаревшими репозиторием и дашбордом из методички аналогичная история). Что происходит и как добавлять логи nginx-ingress, не понятно.

> TODO: Разобраться с отсутствующими данными в ES и ошибками fluent-bit (слайды 25-29, 33-37)

---

Установим Loki и Promtail (в [prometheus-operator.values.yaml](./prometheus-operator.values.yaml) добавлен `grafana.additionalDataSources`):

```
$ helm repo add loki https://grafana.github.io/loki/charts
$ helm repo update
$ helm upgrade --install loki --namespace=observability loki/loki -f ./loki.values.yaml
$ helm upgrade --install promtail loki/promtail --namespace=observability -f ./promtail.values.yaml
$ helm upgrade --install prometheus-operator stable/prometheus-operator -f ./prometheus-operator.values.yaml
```

Не смотря на наличие в конфигмапе датасорца Loki, в графане он отсутствует. Добавим его вручную через WebUI графаны, и он появится в Explore.

Видим, наконец метку `app=nginx-ingress` с "сырыми" логами ингресса, а вот `fluent-bit` "сыплет" ошибками "failed to flush chunk", "could not pack/validate JSON response" и еще какими-то ошибками парсинга. Возвращаемся к задаче с форматированием логов nginx в JSON-формате. Добавим `controller.config` в [values](./nginx.values.yaml), обновим релиз и снова посмотрим на лог в Loki. Теперь каждая запись разбита на осмысленные свойства.

Переходим к визуализации Включаем `serviceMonitor` для ингресса через [values](./nginx.values.yaml). Добавим переменные через импорт дашборда, после чего - панель с запросом

```promql
sum(rate(nginx_ingress_controller_requests{controller_pod=~"$controller",controller_class=~"$controller_class",namespace=~"$namespace",status!~"[4-5].*"}[2m])) / sum(rate(nginx_ingress_controller_requests{controller_pod=~"$controller",controller_class=~"$controller_class",namespace=~"$namespace"}[2m]))
```

Результат - "No data". Добавлять что-то "аналогичным образом", по-видимому, смысла не много.

> TODO: Разобраться с отсутствующими результатами запроса (слайды 43-44, 47)

Обратим внимание на [k8s-event-logger](https://github.com/max-rocket-internet/k8s-event-logger). Делать ничего не будем, просто обратим внимание.

Задание со ⭐ пропускаем...

Готово!