# rabbitmq-kubernetes
Mindbox test task

В качестве решения для деплоя helm chart bitnami/rabbitmq (https://github.com/bitnami/charts/tree/master/bitnami/rabbitmq), официальный заявлен deprecated. Helm, на мой взгляд, самый удобный способ задеплоить стандартное приложение(по крайней мере, после релиза третьей версии), если оно не требует серьезной адаптации под существующую инфраструктуру. Кластер kubernetes развернул на GKE - потому что там самый щедрый триал, по большому счету разницы нет, конфигурация везде будет примерно одинаковая.

4 ноды в кластере - мастер и три воркера, три реплики StatefullSet rabbitmq с учетом, что можно будет добавить еще одну, очереди раскидываются lb. Реализация без HA-плагина, для отказоустойчивости PVC - если нода  или pod отваливается, после запуска возвращается к очередям с диска.

Решил не добавлять HA плагин , так как при заданных нагрузках потребуются жирные воркеры, при этом при увеличении полных реплик или то трех  итоговая производительность мастер ноды снизится до 25% от такой же ноды без бэкапов. Разумеется, это актуально только если нет явного требования хранить данные.
https://www.cloudamqp.com/blog/2017-12-29-part1-rabbitmq-best-practice.html#number-of-nodes-in-your-cluster-clustering-and-high-availability
https://www.rabbitmq.com/ha.html#replication-factor
DaemonSet не подходит по очевидным причинам  - не нужно по одному поду rabbitMQ на каждой ноде. Deployment не подходит, потому что Statefull-приложения плохо реагируют на внезапные рестарты. https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#using-statefulsets

В values.yaml указаны параметры для чарта. В примере не развернут ingress, нет сертификатов и не настроен мониторинг, только экспозятся метрики. В реальности, конечно, должны быть.
lb и pvc дефолтные в GKE и firewall. Реальные требования к ресурсам нод были бы на уровне 5-8 RAM и 4-8 ядрам CPU без hyper-threading при среднем размере сообщения в 512Kb без учета служебной информации
https://groups.google.com/forum/#!topic/rabbitmq-users/PEh9tlZ6A98

В конфиге добавил блок podAntiAffinity, чтобы исключить возможность развертывания нескольких подов на одной ноде - производительности не добавит.
Добавил policy queues expiry на 12 часов. Значение ничем особо не обусловлено, зависит от требованиям к времени хранения данных для консумеров.
Для тестов кластера использовал message generator из https://github.com/GetLevvel/message-simulator. Примеры тестов в scripts.