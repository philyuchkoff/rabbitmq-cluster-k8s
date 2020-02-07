```
enabled_plugins: |
  [rabbitmq_management,rabbitmq_peer_discovery_k8s].

```

  
Добавляем нужные плагины в разрешенные к загрузке. Теперь мы можем использовать автоматический Peer Discovery в K8S.  
  

```
cluster_formation.peer_discovery_backend  = rabbit_peer_discovery_k8s

```

  
Выставляем в качестве backend для peer discovery нужный плагин.  
  

```
cluster_formation.k8s.host = kubernetes.default.svc.cluster.local
cluster_formation.k8s.port = 443

```

  
Указываем адрес и порт, через которые можно достучаться до kubernetes apiserver. Здесь можно указать напрямую ip-адрес, но более красиво будет сделать так.  
  
В namespace default обычно создан service с именем kubernetes, ведущий на k8-apiserver. В разных вариантах установки K8S namespace, имя сервиса и порт могут быть другими. Если что-то в конкретной установке отличается, нужно, соответственно, поправить здесь.  
  
Например, мы столкнулись с тем, что в некоторых кластерах сервис на порту 443, а в некоторых на 6443. Понять, что что-то не так, можно будет в логах старта RabbitMQ, там явно выделен момент подключения по указанному здесь адресу.  
  

```
### cluster_formation.k8s.address_type = ip
cluster_formation.k8s.address_type = hostname

```

  
По умолчанию в примере был указан тип адресации нод кластера RabbitMQ по ip-адресу. Но при перезапуске pod он каждый раз получает новый IP. Сюрприз! Кластер умирает в муках.  
  
Меняем адресацию на hostname. StatefulSet гарантирует нам неизменность hostname в рамках жизненного цикла всего StatefulSet, что нас полностью устроит.  
  

```
cluster_formation.node_cleanup.interval = 10
cluster_formation.node_cleanup.only_log_warning = true

```

  
Поскольку при потере одной из нод мы предполагаем, что она рано или поздно восстановится, отключаем самоудаление кластером недоступных нод. В этом случае, как только нода вернётся в онлайн, она войдёт в кластер без потери своего предыдущего состояния.  
  

```
cluster_partition_handling = autoheal
```

  
Этим параметром определяем действия кластера при потере кворума. Тут стоит просто почитать [документацию по этой теме](http://www.rabbitmq.com/partitions.html) и понять для себя, что ближе к конкретному сценарию использования.  
  

```
queue_master_locator=min-masters
```

  
Определяем выбор мастера для новых очередей. При данной настройке мастером будет выбираться нода с наименьшим количеством очередей, таким образом очереди будут распределяться равномерно по нодам кластера.  
  

```
cluster_formation.k8s.service_name = rabbitmq-internal
```

  
Задаём имя headless сервиса K8s (созданного нами ранее), через который ноды RabbitMQ будут общаться между собой.  
  

```
cluster_formation.k8s.hostname_suffix = .rabbitmq-internal.our-namespace.svc.cluster.local
```

  
Важная штука для работы адресации в кластере по hostname. FQDN пода K8s формируется как короткое имя (rabbitmq-0, rabbitmq-1) + суффикс (доменная часть). Здесь мы и указываем этот суффикс. В K8S он выглядит как **.<имя сервиса>.<имя namespace>.svc.cluster.local**  
  
kube-dns без какой-либо дополнительной настройки резолвит имена вида rabbitmq-0.rabbitmq-internal.our-namespace.svc.cluster.local в ip-адрес конкретного пода, что и делает возможной всю магию кластеризации по hostname.
