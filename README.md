# Лабораторная работа Docker: докеризация приложения
## Задание:
Цель лабораторной: собрать из исходного когда и запустить в докере рабочее приложение с базой данных (любое опенсорс - Java, python/django/flask, golang).  

1. Образ должен быть легковесным  
2. Использовать базовые легковестные образы - alpine  
3. Вся конфигурация приложения должна быть через переменные окружения  
4. Статика (зависимости) должна быть внешним томом `volume`  
5. Создать файл `docker-compose` для старта и сборки  
6. В `docker-compose` нужно использовать базу данных (postgresql,mysql,mongodb etc.)  
7. При старте приложения должно быть учтено выполнение автоматических миграций  
8. Контейнер должен запускаться от непривилегированного пользователя  
9. После установки всех нужных утилит, должен очищаться кеш

## Выполнение:
1. За основу взято приложение, в котором пользователи могут регистрироваться, проходить аутентификацию и управлять своим балансом (пополнять счёт, резервировать и возвращать средства, выводить историю операций и тд). Приложение написано на golang и использует PostgreSQL в качестве хранилища данных.  
2. Для миграций БД используется golang-migrate/migrate. В директории migrations находятся файлы миграций.
3. Dockerfile содержит инструкции для сборки образа. Здесь представлена multi-stage сборка:  
   1 этап:  
```
FROM golang:alpine as modules
COPY go.mod go.sum /modules/
WORKDIR /modules
RUN go mod download && go clean -modcache
```  
Здесь происходит управление модулями go, благодаря этому ускоряются последующие сборки.  
   2 этап:  
```
FROM golang:alpine as builder
COPY --from=modules /go/pkg /go/pkg
COPY . /app
WORKDIR /app
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -tags migrate -o /bin/app ./cmd/app \
    && go clean -cache -testcache -modcache -i -r
```
   Здесь собирается приложение.  
   3 этап:
```
FROM scratch
COPY --from=builder /app/config /config
COPY --from=builder /app/migrations /migrations
COPY --from=builder /bin/app /app
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
CMD ["/app"]
```
   Здесь запускается приложение.  
5. Конфигурация проекта содержится в файле docker-compose.yaml.  
6. Переменные окружения, через которые осуществляется конфигурация, находятся в файле .env.   

## Запуск:
Создание контейнеров и их запуск в фоновом режиме:  
```
docker-compose up --build -d
```
Ниже на скриншоте показано создание контейнеров и работа с приложением (регистрация двух пользователей (обе со второй попытки) и авторизация одного пользователя):  
![alt-текст][logo]

[logo]: https://github.com/ulyanovaktrn/Docker_lab/blob/main/screenshots/app_docker.png  
Просмотр запущенных в данный момент контейнеров:
```
docker ps
```
Флаг -a добавит к выводу информацию о тех контейнерах, которые были остановлены, завершены или не были запущены успешно (было полезно для отладки).  
![alt-текст][logo1]

[logo1]: https://github.com/ulyanovaktrn/Docker_lab/blob/main/screenshots/ps-a.png  
Просмотр логов, сгенерированных конкретным контейнером:  
```
docker logs <container_name_or_id>
```
![alt-текст][logo2]

[logo2]: https://github.com/ulyanovaktrn/Docker_lab/blob/main/screenshots/logs.png  
Отображение списка локально сохранённых образов:
```
docker images
```
С помощью этой команды можно узнать размер нашего образа (40.3MB).  
![alt-текст][logo3]

[logo3]: https://github.com/ulyanovaktrn/Docker_lab/blob/main/screenshots/image_size.png
Остановка контейнеров:
```
docker-compose stop
```
![alt-текст][logo4]

[logo4]: https://github.com/ulyanovaktrn/Docker_lab/blob/main/screenshots/stop.png    
Запуск контейнеров:
```
docker-compose start
```
![alt-текст][logo5]

[logo5]: https://github.com/ulyanovaktrn/Docker_lab/blob/main/screenshots/start.png  
Завершение работы всех контейнеров с удалением всех ресурсов, связанных с ними:
```
docker-compose down -v
```
![alt-текст][logo6]

[logo6]: https://github.com/ulyanovaktrn/Docker_lab/blob/main/screenshots/down.png  
