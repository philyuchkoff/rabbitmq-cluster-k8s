# RabbitMQ кластер в k8s

## Задача:
> Есть виртуальная машина с менеджером очередей RabbitMQ, несколько тысяч очередей и миллионы сообщений в день.
> 
> Задача: Хотим получить отказоустойчивость и масштабируемость.


#### Описание директорий:

**RBAC** - права и роли; 
**Volume** - хранилище; 
**Services** - сервисы.

## Итог:

В результате получится кластер RabbitMQ, равномерно распределяющий очереди по нодам и устойчивый к проблемам в среде выполнения.

При недоступности одной из нод кластера, очереди, содержащиеся на ней, перестанут быть доступны, всё остальное продолжит работу. Как только нода вернётся в строй, она вернётся в кластер, и очереди, для которых она была Master'ом, снова станут работоспособными с сохранением всех содержащихся в них данных (если не сломалось персистентное хранилище, разумеется). Все эти процессы проходят полностью автоматически и не требуют вмешательства.

### Настраиваем HA

Требуется полное зеркалирование всех содержащихся в кластере данных. Это нужно, чтобы в ситуации, когда хотя бы одна нода кластера работоспособна, с точки зрения прикладного приложения всё продолжало работать. *Этот момент никак не связан именно с K8s.*  
  
Для включения полного HA необходимо в RabbitMQ dashboard на вкладке _Admin -> Policies_ создать Policy. Имя произвольное, Pattern пустой (все очереди), в Definitions добавить два параметра: _ha-mode: all_, _ha-sync-mode: automatic_.

После этого все создаваемые в кластере очереди будут находиться в режиме High Availability: при недоступности Master-ноды новым мастером автоматически будет выбираться один из Slave’ов. А данные, поступающие в очередь, будут зеркалироваться на все ноды кластера. Что, собственно, и требовалось получить.

http://www.rabbitmq.com/ha.html


## Полезная литература:

  

-   [Репозиторий RabbitMQ Peer Discovery Kubernetes Plugin](https://github.com/rabbitmq/rabbitmq-peer-discovery-k8s/)
-   [Пример конфигурации для деплоя RabbitMQ в K8S](https://github.com/rabbitmq/rabbitmq-peer-discovery-k8s/tree/master/examples/k8s_statefulsets)
-   [Описание принципов формирования кластера, механизма Peer Discovery и плагинов для него](http://www.rabbitmq.com/cluster-formation.html)
-   [Эпичная дискуссия о правильной настройке hostname-based discovery](https://groups.google.com/forum/#!msg/rabbitmq-users/wuOfzEywHXo/k8z_HWIkBgAJ)
-   [Руководство по кластеризации RabbitMQ](http://www.rabbitmq.com/clustering.html)
-   [Описание split-brain проблем при кластеризации и способов их решения](http://www.rabbitmq.com/partitions.html)
-   [High Availability Queues в RabbitMQ](http://www.rabbitmq.com/ha.html)
-   [Настройка Policies](http://www.rabbitmq.com/parameters.html#policies)

### Helm
Есть готовый [helm chart](https://github.com/helm/charts/tree/master/stable/rabbitmq-ha)
