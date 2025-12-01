# Домашнее задание к занятию «Запуск приложений в K8S»

### Грибанов Антон. FOPS-31

### Цель задания

В тестовой среде для работы с Kubernetes, установленной в предыдущем ДЗ, необходимо развернуть Deployment с приложением, состоящим из нескольких контейнеров, и масштабировать его.

------

### Чеклист готовности к домашнему заданию

1. Установленное k8s-решение (например, MicroK8S).
2. Установленный локальный kubectl.
3. Редактор YAML-файлов с подключённым git-репозиторием.

------

### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

1. [Описание](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) Deployment и примеры манифестов.
2. [Описание](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/) Init-контейнеров.
3. [Описание](https://github.com/wbitt/Network-MultiTool) Multitool.

------

### Задание 1. Создать Deployment и обеспечить доступ к репликам приложения из другого Pod

1. Создать Deployment приложения, состоящего из двух контейнеров — nginx и multitool. Решить возникшую ошибку.
2. После запуска увеличить количество реплик работающего приложения до 2.
3. Продемонстрировать количество подов до и после масштабирования.
4. Создать Service, который обеспечит доступ до реплик приложений из п.1.
5. Создать отдельный Pod с приложением multitool и убедиться с помощью `curl`, что из пода есть доступ до приложений из п.1.


### Решение:

1. Создал deployment.yaml со следующим содержимым:

```bash
   apiVersion: apps/v1
kind: Deployment
metadata:
  name: multitool
spec:
  replicas: 1
  selector:
    matchLabels:
      app: multitool
  template:
    metadata:
      labels:
        app: multitool
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
      - name: multitool
        image: wbitt/network-multitool:latest
        ports:
        - containerPort: 8080
```

2. Но при проверке статуса получаем ошибку:

![k8s_03](https://github.com/Qshar1408/k8s_03/blob/main/img/k8s_03_001.png)

3. В логах видим следующее:
   
![k8s_03](https://github.com/Qshar1408/k8s_03/blob/main/img/k8s_03_002.png)

4. В контейнере multitool еще один экзепляр nginx занимает порт 80, который уже используется в другом контейнере, так как на одном порту пытаеются запуститься сразу 2 веб сервера, то возникает ошибка. Добавил в deployment.yaml ENV со следующим значением:

```bash
 env:
        - name: HTTP_PORT
          value: "7080"
```

5. Финальный вариант:

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multitool
spec:
  replicas: 2
  selector:
    matchLabels:
      app: multitool
  template:
    metadata:
      labels:
        app: multitool
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
      - name: multitool
        image: wbitt/network-multitool:latest
        ports:
        - containerPort: 8080
        env:
        - name: HTTP_PORT
          value: "7080"
```

6. Проверяем работу:

![k8s_03](https://github.com/Qshar1408/k8s_03/blob/main/img/k8s_03_003.png)

7. Затем в deployment.yaml изменил значение replicas на 2, тем самым увеличив количество реплик приложения до двух:

![k8s_03](https://github.com/Qshar1408/k8s_03/blob/main/img/k8s_03_004.png)

8. Далее, создал deployment-svc.yaml с содержимым для создания сервиса для доступа к репликам приложений:

```bash
apiVersion: v1
kind: Service
metadata:
  name: deployment-svc
spec:
  selector:
    app: multitool
  ports:
  - name: for-nginx
    port: 80
    targetPort: 80
  - name: for-multitool
    port: 7080
    targetPort: 7080
```

![k8s_03](https://github.com/Qshar1408/k8s_03/blob/main/img/k8s_03_005.png)

9. Далее, создал отдельный под для приложения multitool multitool-app.yaml:

```bash
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: multitool
  name: multitool-app
  namespace: default
spec:
  containers:
  - name: multitool
    image: wbitt/network-multitool
    ports:
    - containerPort: 8080
    env:
      - name: HTTP_PORT
        value: "7080"
```

![k8s_03](https://github.com/Qshar1408/k8s_03/blob/main/img/k8s_03_006.png)

10. Проверяю изнури нового пода multitool-app доступ до раннее созданных приложений:

![k8s_03](https://github.com/Qshar1408/k8s_03/blob/main/img/k8s_03_007.png)

![k8s_03](https://github.com/Qshar1408/k8s_03/blob/main/img/k8s_03_008.png)

![k8s_03](https://github.com/Qshar1408/k8s_03/blob/main/img/k8s_03_009.png)


------

### Задание 2. Создать Deployment и обеспечить старт основного контейнера при выполнении условий

1. Создать Deployment приложения nginx и обеспечить старт контейнера только после того, как будет запущен сервис этого приложения.
2. Убедиться, что nginx не стартует. В качестве Init-контейнера взять busybox.
3. Создать и запустить Service. Убедиться, что Init запустился.
4. Продемонстрировать состояние пода до и после запуска сервиса.


### Решение:

1. Создал deployment-init.yaml:

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-app
  name: nginx-app
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
      initContainers:
      - name: init-nginx-svc
        image: busybox
        command: ['sh', '-c', 'until nslookup nginx-svc.default.svc.cluster.local; do echo waiting for nginx-svc; sleep 5; done;']
```

2. При проверке nginx не запускается:

![k8s_03](https://github.com/Qshar1408/k8s_03/blob/main/img/k8s_03_010.png)

3. Создал Service nginx-svc.yaml:

```bash
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  selector:
    app: nginx
  ports:
  - name: http-port
    port: 80
    protocol: TCP
    targetPort: 80
```

После него init запустился:

![k8s_03](https://github.com/Qshar1408/k8s_03/blob/main/img/k8s_03_011.png)



------

### Правила приема работы

1. Домашняя работа оформляется в своем Git-репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд `kubectl` и скриншоты результатов.
3. Репозиторий должен содержать файлы манифестов и ссылки на них в файле README.md.
