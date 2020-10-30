# Лабораторная работа №3: CI/CD и деплой приложения в Heroku

В данной лабораторной работе необходимо реализовать простой CI/CD для деплоя приложения в Heroku из репозитория.

Целью лабораторной работы является знакомство с CI/CD и его реализацией на примере Travis CI и Heroku. 

## Задание

1. Реализовать непрерывную интеграцию (Continuous Integration) посредством Travis CI для своего приложения.

2. Реализовать непрерывную доставку (Continuous Deployment) обновлений в Heroku посредством Travis CI и DockerHUB. 

## Форма отчетности

1 В самом низу README.md из лабораторной №1 дополнить информацией:
* Название лабораторной работы.
* Цель лабораторной работы.

2. Привести ссылку на развернутое приложение на платформе Heroku.

3. Прикрепить Travis CI badge на странице репозитория для отображения статуса сборки.

## Инструкции к выполнению задания

1. Настроить приложение, так чтобы оно при старте проверяло созданы ли необходимые таблицы, если нет, то таблицы должны создаваться. Для этого прописать в application.properties:

        spring.jpa.hibernate.ddl-auto=update

2. Убедиться что путь к БД в том же файле указывает на localhost:

        spring.datasource.url=jdbc:postgresql://localhost:5432/postgres

3. В прошлой лабораторной мы в классе CommandLineAppStartupRunner тестрировали работу репозитория. Теперь нам этот класс не нужен - его можно удалить. В противном случае могут возникнут ошибки во время старта приложения, из-за попытки получить запись из пустой базы данных.

4. Зарегистрироваться на http://travis-ci.com и подготовить в папке с приложеннием конфиг .travis.yml:

        dist: trusty # репозиторий внутри travis
        jdk: oraclejdk8
        language: java
        services:
          - postgresql
          - docker

5. Подключить репозиторий со своим приложением к Travis CI.

6. Запушить изменения в репозиторий, после чего убедиться что Travis CI их обнаружил и сделал успешную сборку.

7. Зарегисрироваться на http://heroku.com и создать там приложение с любым именем, которое потом будет отображаться в url: 

        https://имя_приложения.herokuapp.com

8. Перейти во вкладку Resources и добавить Add-on: Heroku Postgres.

9. Подключиться к созданной базе из IntelliJ IDEA. Для этого на странице, где был добавлен Add-on, кликнуть на появившуюся иконку Heroku Postgres, затем нажать вкладку Settings, после чего кнопку View Credentials.

Необходимо использовать эти параметры доступа для подключения к БД.

10. Загрузить в БД схему данных из schema.sql и сами данные из data.sql.

11. Зарегистрироваться на http://hub.docker.com и создать там свой репозиторий для докер образов, например my-docker-repo.

12. Дополнить файл .travis.yml параметрами, которые будут описывать:

* сборку проекта;
* вход в DockerHUB;
* сборку докер-образа;
* установку тега на образ;
* отправку собранного образа в репозиторий my-docker-repo;
* деплой образа в Heroku.

Тогда .travis.yml будет выглядеть следующим образом:

        dist: trusty
        jdk: oraclejdk8
        language: java
        services:
          - postgresql
          - docker
        env:
          global:
            - secure: "T9lgJZC8q57+bE5MCWQjnzkIfOe3e9CzhxA6QA="
            - secure: "k4Yc2Wvclhqc79qQYMY8JBYVakDM2vwmfGRC28="
            - secure: "Ajcf4DIdVNLQ8jehT+rZ2fvUhHonszOECV6XTY="
            - COMMIT=${TRAVIS_COMMIT::7}
        
        script:
          - ./mvnw clean install -B
        
        after_success:
          - docker login -u $DOCKER_USER -p $DOCKER_PASS
          - export TAG=`if [ "$TRAVIS_BRANCH" == "master" ]; then echo "latest"; else echo "$TRAVIS_BRANCH"; fi`
          - export IMAGE_NAME=myapi/my-docker-repo
          - docker build -t $IMAGE_NAME:latest .
          - docker tag $IMAGE_NAME:latest $IMAGE_NAME:$TAG
          - docker push $IMAGE_NAME:$TAG
        
        deploy:
          provider: heroku
          api_key: $HEROKU_API_KEY
          app: имя_приложения

Примечение: в 3-х строчках в секции env.global находятся зашифрованные параметры и их значения:

HEROKU_API_KEY="ваш зашифрованный ключ api"
DOCKER_USER="ваш зашифрованный юзернейм в DockerHUB"
DOCKER_PASS="ваш зашифрованный пароль в DockerHUB"

Ключ HEROKU_API_KEY находится в Account Settings на http://heroku.com

Каждый из параметров прежде чем вставить в подсекции secure необходимо зашифровать с помощью консольной утилиты travis. Как ее установить можно найти [здесь](https://github.com/travis-ci/travis.rb#installation).

После установки travis, откройте консоль и перейдите в папку с Вашим приложением. Введите travis login --pro, появится приглашение, где введите Ваши логин и пароль от репозитория git.

Теперь можно шифровать параметры. Примеры команд для шифрования:

        travis encrypt HEROKU_API_KEY="aedbcbcb-2c66-40fa-b6ae-532a25c922da" --com
        travis encrypt DOCKER_USER="username" --com
        travis encrypt DOCKER_PASS="password" --com

Полученные защифрованные значения замените в строчках элментов secure в упомянутой ранее секции env.global.

13. Запушьте изменения и проследите, чтобы сборка в Travis CI прошла успешно и Ваше приложение задеплоилось в Heroku.

14. Убедитесь, что ендпоинты вашего приложения доступны по адресу: https://имя_приложения.herokuapp.com/api/v1/

* [Запись лекции-вебинара по CI/CD и деплою приложения в Heroku]()
