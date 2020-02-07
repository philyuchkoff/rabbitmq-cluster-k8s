
```
serviceName: rabbitmq-internal
```

  
Прописываем имя headless-сервиса, через который общаются поды в StatefulSet.  
  

```
replicas: 3
```

  
Задаём количество реплик в кластере. У нас оно равно числу рабочих нод K8s.  
  

```
annotations:
        scheduler.alpha.kubernetes.io/affinity: >
            {
              "podAntiAffinity": {
                "requiredDuringSchedulingIgnoredDuringExecution": [{
                  "labelSelector": {
                    "matchExpressions": [{
                      "key": "app",
                      "operator": "In",
                      "values": ["rabbitmq"]
                    }]
                  },
                  "topologyKey": "kubernetes.io/hostname"
                }]
              }
            }
```

  
При падении одной из нод K8s statefulset стремится сохранить количество экземпляров в наборе, поэтому создаёт по нескольку подов на одной и той же ноде K8s. Это поведение совершенно нежелательно и в принципе бессмысленно. Поэтому мы прописываем anti-affinity правило для подов из statefulset. Правило делаем жестким (Required), чтобы kube-scheduler не мог его нарушать при планировании подов.  
  
Суть проста: планировщику запрещено размещать (в пределах namespace) более одного пода с тегом _app:rabbitmq_ на каждой ноде. Ноды различаем по значению метки _kubernetes.io/hostname_. Теперь если по какой-то причине число работающих нод K8S меньше, чем требуемое количество реплик в StatefulSet, новые реплики не будут создаваться, пока снова не появится свободная нода.  
  

```
serviceAccountName: rabbitmq
```

  
Прописываем ServiceAccount, под которым работают наши поды.  
  

```
image: rabbitmq:3.7
```

  
Образ RabbitMQ совершенно стандартный и берётся с docker hub, не требует никакой пересборки и доработки напильником.  
  

```
- name: rabbitmq-data
    mountPath: /var/lib/rabbitmq/mnesia

```

  
Персистентные данные у RabbitMQ хранятся в /var/lib/rabbitmq/mnesia. Здесь мы монтируем наш Persistent Volume Claim в эту папку, чтобы при перезапуске подов/нод или даже всего StatefulSet данные (как служебные, в том числе о собранном кластере, так и пользовательские) оставались в целости и сохранности. Можно встретить некоторые примеры, когда персистентной делают папку /var/lib/rabbitmq/ целиком. Мы пришли к выводу, что это не самая лучшая идея, так как при этом начинает запоминаться и вся информация, заданная конфигами Rabbit. То есть для того, чтобы изменить что-то в конфигурационном файле, требуется почистить персистентное хранилище, что очень неудобно в эксплуатации.  
  

```
          - name: HOSTNAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
          - name: RABBITMQ_USE_LONGNAME
            value: "true"
          - name: RABBITMQ_NODENAME
            value: "rabbit@$(HOSTNAME).rabbitmq-internal.$(NAMESPACE).svc.cluster.local"

```

  
Этим набором переменных окружения мы, во-первых, говорим RabbitMQ использовать в качестве идентификатора членов кластера FQDN-имя, а во-вторых, задаём формат этого имени. Формат описывался ранее при разборе конфига.  
  

```
- name: K8S_SERVICE_NAME
            value: "rabbitmq-internal"
```

  
Имя headless сервиса для общения членов кластера.  
  

```
- name: RABBITMQ_ERLANG_COOKIE
            value: "mycookie"
```

  
Содержимое Erlang Cookie должно быть одинаковым на всех нодах кластера, нужно прописать ваше собственное значение. Нода с отличающимся cookie не сможет войти в кластер.  
  

```
volumes:
        - name: rabbitmq-data
          persistentVolumeClaim:
            claimName: rabbitmq-data
```

  
Определяем подключаемый том из созданного ранее Persistent Volume Claim.
