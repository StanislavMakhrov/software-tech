# Лабораторная работа №2: создание кластера Kubernetes и деплой приложения

В данной лабораторной работе необходимо развернуть кластер Kubernetes на локальной рабочей станции посредством MiniKube.

Целью лабораторной работы является знакомство с кластерной архитектурой на примере Kubernetes, а также деплоем приложения в кластер. 

## Задание

1. Развернуть локально кластер на Kubernetes с использованием MiniKube.

2. Произвести деплой приложения в кластер. У приложения должны быть ендпоинты, соответствующие его предметной области, а также ендпоинт, где будет отображаться hostname.

## Форма отчетности

1. Сделать в github/gitlab Markdown-страницу, где указать:
* Название дисциплины.
* Название лабораторной работы.
* ФИО и группу.
* Цель лабораторной работы.
* Манифесты deployment.yaml и service.yaml в тексте страницы.
* Скриншты вывода команды консоли с шага 3.3 на фоне рабочего стола.
* Скриншоты графического интерфейса с шага 3.5, где видны поды.
* 30 секундное видео с обзором созданного кластера и вашимментариями.

## Инструкции к выполнению задания

* [Запись лекции-вебинара по созданию кластера Kubernetes и деплою приложения](https://drive.google.com/file/d/1vs6G7x2NwRoEpS12TrhvG3V-wbv7paQ3/view?usp=sharing)

### 1. Развертывание локального кластера на Kubernetes с использованием MiniKube

1.1. Установите MiniKube, выполнить 1 и 2 шаг из инструкции https://minikube.sigs.k8s.io/docs/start/

1.2. Установите kubectl https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/

1.3. Убедитесь, что kubectl работает и произведите осмотрите кластера:

        kubectl get node
        kubectl get po
        kubectl get po -A
        kubectl get svc

1.4. Установите графический интерфейс Dashboard https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/ - 
необходимо выполнить шаги Deploying the Dashboard UI и Accessing the Dashboard UI. В последнем не забудьте кликнуть по ссылке creating a sample user и выполнить там инструкции.

1.5. Кластер готов! Шпаргалка по командам: https://kubernetes.io/ru/docs/reference/kubectl/cheatsheet/

### 2. Деплой приложения

2.1. Создать манифест Deployment и сохранить в файл, например deployment.yaml:

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

2.2. Создать манифест Service и сохранить в файл, например deployment.yaml

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

2.3. Решение проблемы с доступностью Registry с докер-образами.

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
                    
2.4. Решение проблемы с обращением из MiniKube к приложениям, либо БД на localhost:

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
              - ip: "192.168.49.1" # The IP of localhost from MiniKube
                hostnames:
                - postgres.local

2.5. Манифесты deployment.yaml и service.yaml лучше всего сохранить в одной папке, где не будет ничего лишнего и потом выполнить команду для их применения:

        kubectl apply -f .
        
2.6. После выполнения проверить статус подов:

        kubectl get po
        NAME                             READY   STATUS    RESTARTS   AGE
        my-deployment-7c8c985d48-87p4s   1/1     Running   1          12h
        my-deployment-7c8c985d48-z7zbq   1/1     Running   1          12h
        
2.7. Посмотреть логи конкретного пода:

        kubectl logs my-deployment-7c8c985d48-87p4s

2.8. Попробовать обратиться к приложению в MiniKube:

        http://192.168.49.2:31317/api/v1/status

где 192.168.49.2 - адрес MiniKube, который мы ранее смотрели в ~/.kube/config.

2.9. Попробовать обратиться к существующим ендпоинтам приложения.

### 3. Увеличение количества реплик до 10 и проверка отображение hostname.

3.1. В манифесте deployment.yaml изменить количество реплик до 10:

spec.replicas: 10

3.2. Применить изменения:

        kubectl apply -f deployment.yaml

3.3. После выполнения проверить количество подов с помощью команды:

        kubectl get po
        
3.4. Попробовать обращаться множество раз к ендпоинту, где отображается hostname http://192.168.49.2:31317/api/v1/status и наблюдать как он меняется - в зависимости от пода, куда балансировщик распределил запрос.

3.5. Произвести осмотр подов в графическом интерфейсе:

http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/

Не забудьте, что должна быть активирована прокси с шага 1.4, где устанавливали dashboard с UI.