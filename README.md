# Описание проекта

Данное приложение создано для конвертации аудиофайлов из формата wav в формат mp3. Используются Flask, SQLAlchemy,
PostgreSQL, Docker. Возможно развернуть путем непосредственного запуска приложения, а также путем разворачивания
контейнера через docker-compose. Для работы необходимо наличие библиотеки ffmpeg.

### URL-адреса

Для отправки запросов доступны следующие URL-адреса:

- <code>/user/add/</code>
- <code>/record/</code>

#### /user/add/

Добавляет нового пользователя и в ответе отправляет сформированный ID и уникальный идентификационный токен.
Чтобы осуществить успешный запрос, нужно отправить в теле запроса JSON вида - <code>{"name": "string"}</code>,
где <code>"string"</code> это желаемое имя пользователя. В ответе будет содержаться информация с токеном

Пример запроса:
<pre>
curl 
-X POST 
-H "Content-Type: application/json" 
-d '{"name": "nikita"}' 
http://127.0.0.1:5000/user/add/
</pre>

Пример ответа на запрос выше:
<pre>
{
"credentials": {"id": 1, 
                "token": "165a8e59-0648-4829-b902-2ec6a617a3c8"}
}
</pre>

#### /record/

Конвертирует отправленную аудиозапись в формат .mp3 и в ответе Пользователю содержит ссылку на скачивание нового файла.
Чтобы осуществить успешный запрос, нужно отправить данные в формате multipart/form-data, содержащие ID
зарегистрированного пользователя, персональный идентификационный токен, а также файл, который нужно конвертировать.
Важно сохранить названия user_id, user_token, audio.

Пример запроса:
<pre>
curl 
-X POST 
-H "Content-Type: multipart/form-data" 
-F "user_id"="1" 
-F "user_token"="52d40754-2615-40df-93c4-40d2d36c344c" 
-F "audio=@/"abs_path"/example.wav;type=audio/wave" 
http://127.0.0.1:5000/record/
</pre>

После успешной конвертации аудиофайла, сервер пришлет в ответе ссылку на скачивание файла:

<code>
http://127.0.0.1:5000/record/?id=423b9345-0a21-4025-b7ae-45f08f46fff1&user=1
</code>

Можно открыть эту ссылку в браузере и файл скачается самостоятельно либо можно отправить запрос через терминал

Пример запроса:

<code>
curl -o file.mp3 "http://127.0.0.1:5000/record/?id=423b9345-0a21-4025-b7ae-45f08f46fff1&user=1"
</code>, где 
<code>file.mp3</code> после флага <code>-о</code> означает название файла, с которым нужно скачать файл

# Настройка работы путем контейнеризации через docker-compose

## Подготовка перед запуском

1. Если Docker и docker-compose не установлены, то необходимо, руководствуясь документацией, осуществить установку
   дистрибутивов:
   https://docs.docker.com/engine/install/ubuntu/
   https://docs.docker.com/compose/install/linux/

2. Нужно создать папку на хостовой машине (в этой папке будет храниться копия базы данных из контейнера). При изменении
   директории, надо внести соответствующие правки в файл docker-compose на строке 34

```text
/var/lib/postgresql/wav_mp3_converter/
```

3. Дать ей права на запись пользователю, под которым запускается Docker

```text
sudo chown -R $USER:$USER /var/lib/postgresql/wav_mp3_converter/
```

Пример:

```text
sudo chown -R orlovnikita:orlovnikita /var/lib/postgresql/wav_mp3_converter/
```

4. Открыть в терминале папку music_service, содержащую текущий проект и выполнить две последовательные команды.
   Ожидание сборки образа контейнера составит примерно 1-2 минуты в зависимости от скорости интернета. Результатом
   станет
   запущенное flask приложение. Обратиться к нему можно через URL-адрес http://127.0.0.1:5000/user/add/, например

```text
docker-compose build
docker-compose up
```

P.S.
Если после первоначального запуска возникнет ошибка
<code>sqlalchemy.exc.OperationalError: TCP/IP connections on port 5432?</code>
нужно перезапустить контейнер

## Просмотр базы данных из контейнера

1. Посредством команды docker ps смотрим ID действующей контейнера с базой данных postgres и далее подключаемся к
   нему через команду, где bf5f07cb3611 - это ID

```text
docker exec -it bf5f07cb3611 bash
```

2. Подключаемся к базе данных, где "DB_USER" и "DB_NAME" это параметры из файла docker-compose.yml

```text
psql -U "DB_USER" -d "DB_NAME"
```

## Примечание

Локальное хранение копии базы данных приводит к тому, что в случае каких-либо изменений в
переменных окружения после первого запуска контейнера могут возникать ошибки с доступом к БД

# Настройка работы проекта на операционной среде Linux без использования docker-compose

## Установка виртуального окружения и библиотек/пакетов

1. Создаем виртуальное окружение. Данная команда создаст папку venv в папке, откуда была выполнена команда, и в
   терминале появится запись *(venv)*

```text
python3 -m venv venv
```

2. Устанавливаем все требуемые для работы библиотеки. Для этого необходимо в терминале (находясь в папке, в которой
   расположен файл requirements.txt) выполнить следующую команду:

```text
pip install -r requirements.txt
```

3. Устанавливаем необходимую для непосредственной конвертации утилиту ffmpeg

```text
sudo apt-get install -y ffmpeg
```

## Подготовка базы данных PostgreSQL и создание виртуального окружения переменных

1. Подключаемся к терминалу psql от имени пользователя по умолчанию postgres

```text
sudo -u postgres psql
```

2. Создаем базу данных, создаем нового пользователя, меняем некоторые настройки (кодировка, чтение транзакций и
   часовой пояс), открываем для созданного пользователя все возможности работы с новой БД

```postgresql
CREATE DATABASE db_name;
CREATE USER username WITH PASSWORD 'password';
ALTER ROLE username SET client_encoding TO 'utf8';
ALTER ROLE username SET default_transaction_isolation TO 'read committed';
ALTER ROLE username SET timezone TO 'Europe/Moscow';
GRANT ALL PRIVILEGES ON DATABASE db_name TO username;
```

3. В директории вместе с текущим файлом расположен файл ".env.template". Нужно переименовать его в ".env" и переместить
   в одну директорию вместе с файлом settings.py, а потом заполнить по принципу:

```dotenv
DEBUG=True_or_False
#в соответствии с созданными пользователем и БД
DB_NAME=db_name
DB_USER=username
DB_PASSWORD=password
DB_HOST=localhost
```
