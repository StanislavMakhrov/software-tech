# Лабораторная работа №2: создание кластера Kubernetes и деплой туда приложения

## Развертывание локального кластера на Kubernetes с использованием MiniKube

1. Установите MiniKube, выполник 1 и 2 шаг из инструкции https://minikube.sigs.k8s.io/docs/start/
2. Установите kubectl https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/
3. Убедитесь, что kubectl работает и произведите осмотрите кластера:

        kubectl get node
        kubectl get po
        kubectl get po -A
        kubectl get svc

4. Установите графический интерфейс Dashboard https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/ - 
необходимо выполнить шаги Deploying the Dashboard UI и Accessing the Dashboard UI. В последнем не забудьте кликнуть по ссылке creating a sample user и выполнить там инструкции.

5. Кластер готов! Шпаргалка по командам: https://kubernetes.io/ru/docs/reference/kubectl/cheatsheet/

## Деплой приложения

1. Создать манифест Deployment и сохранить в файл, например deployment.yaml:

        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: my-deployment
        spec:
          replicas: 2
          selector:
            matchLabels:
              app: my-app
          strategy:
            rollingUpdate:
              maxSurge: 1
              maxUnavailable: 1
            type: RollingUpdate
          template:
            metadata:
              labels:
                app: my-app
            spec:
              containers:
                - image: myapi:latest
                  name: myapi
                  ports:
                    - containerPort: 8080

Название деплоймента - metadata.name: my-deployment.

Количество реплик здесь задано spec.replicas: 2.

Селектор для сопоставления приложения задан как spec.selector.app: my-app.

Создаваемые конитейнеры с указанием портов, на которых они работают, указаны в секции containers.

Подробнее прочитать можно здесь: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/

2. Создать манифест Service и сохранить в файл, например deployment.yaml

        apiVersion: v1
        kind: Service
        metadata:
          name: my-service
        spec:
          type: NodePort
          ports:
            - nodePort: 31317
              port: 8080
              protocol: TCP
              targetPort: 8080
          selector:
            app: my-app
            
Название сервиса: metadata.name: my-service.

Тип spec.type: NodePort.

Nodeport - это такой тип сервиса, который позволяет входящим соединениям на ноде MiniKube достигать подов, где расположены сами приложения.
Это нужно для того, чтобы мы могли получить доступ к приложению с консоли и из браузера на localhost.
nodePort - может быть задано из диапазона 30000-32767, и каждая нода кластера (фактически одна в Minikube) будет сопоставлять этот порт на порт приложения.
port

Service может сопоставить любой входящий порт port с targetPort. По умолчанию и для удобства targetPort имеет то же значение, что и поле port.

Подрообнее прочитать можно здесь: https://kubernetes.io/ru/docs/tutorials/kubernetes-basics/expose/expose-intro/

3. Решение проблемы с доступностью Registry с докер-образами.

Для того, чтобы MiniKube видел образы docker, необходимо создавать docker-образы внутри MiniKube. Для обеспечения этого, выполните в консоли команду:

    eval $(minikube docker-env)
    
Если у Вас Windows: https://stackoverflow.com/questions/48376928/on-windows-setup-how-can-i-get-docker-image-from-local-machine

Пересоберите докер образ:

    docker build . -t myapi:latest

Пояснение: https://stackoverflow.com/questions/52310599/what-does-minikube-docker-env-mean

В манифесте деплоймента также укажите параметр imagePullPolicy: Never.

Должно получиться так:

        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: my-deployment
        spec:
          replicas: 2
          selector:
            matchLabels:
              app: my-app
          strategy:
            rollingUpdate:
              maxSurge: 1
              maxUnavailable: 1
            type: RollingUpdate
          template:
            metadata:
              labels:
                app: my-app
            spec:
              containers:
                - image: myapi:latest
                  # https://medium.com/bb-tutorials-and-thoughts/how-to-use-own-local-doker-images-with-minikube-2c1ed0b0968
                  imagePullPolicy: Never 
                  name: myapi
                  ports:
                    - containerPort: 8080
                    
4. Решение проблемы с обращением из MiniKube к приложениям, либо БД на localhost:

Узнайте на какой адрес шлет запросы MiniKube для доступа к localhost. Для этого зайдите в ~/.kube/config и посмотрите там адрес кластера.
Например, в файле может быть указано следующее:

        apiVersion: v1
        clusters:
        - cluster:
            certificate-authority: /home/colddeath/.minikube/ca.crt
            server: https://192.168.49.2:8443
          name: minikube
          
Поскольку хост кластера на 192.168.49.2, то на localhost скорее всего можно будет обратиться через 192.168.49.1.
Для того, чтобы проверить доступность localhost изнутри Minikube, зайдите внутрь Minikube путем выполнения в консоли:

        minikube ssh
        
Далее выполните curl к какому-нибудь сервису на 192.168.49.1:

        curl http://192.168.49.1:8080/api/v1/status
        
Если запрос выпоолнился, успешно, то адрес верный.

Теперь необходимо задать для 192.168.49.1 синоним, например postgres.local в манифесте деплоймента.
Через этот синоним будет удобно обращаться на localhost. Тогда деплоймент будет выглядеть следующим образом:

        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: my-deployment
        spec:
          replicas: 2
          selector:
            matchLabels:
              app: my-app
          strategy:
            rollingUpdate:
              maxSurge: 1
              maxUnavailable: 1
            type: RollingUpdate
          template:
            metadata:
              labels:
                app: my-app
            spec:
              containers:
                - image: myapi:latest
                  # https://medium.com/bb-tutorials-and-thoughts/how-to-use-own-local-doker-images-with-minikube-2c1ed0b0968
                  # указыаает на то, что образы нужно брать только из локального registry. В продакшене никогда не использовать
                  imagePullPolicy: Never 
                  name: myapi
                  ports:
                    - containerPort: 8080
              hostAliases:
              - ip: "192.168.49.1" # The IP of your VM
                hostnames:
                - postgres.local

5. Манифесты deployment.yaml и service.yaml лучше всего сохранить в одной папке, где не будет ничего лишнего и потом выполнить команду для их применения:

        kubectl apply -f .
        
6. После выполнения проверить статус подов:

        kubectl get po
        NAME                             READY   STATUS    RESTARTS   AGE
        my-deployment-7c8c985d48-87p4s   1/1     Running   1          12h
        my-deployment-7c8c985d48-z7zbq   1/1     Running   1          12h
        
7. Посмотреть логи конкретного пода:

        kubectl logs my-deployment-7c8c985d48-87p4s

8. Попробовать обратиться к приложения в MiniKube:

        http://192.168.49.2:31317/api/v1/status

где 192.168.49.2 - адрес MiniKube, который мы ранее смотрели в ~/.kube/config.