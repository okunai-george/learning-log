# Контейнеризация с Docker
Ссылка на курс https://practicum.yandex.ru/learn/yc-devops-container/

## Отличия образа от контейнера
Образ &mdash; упакованный набор файлов приложения или сервиса.

Контейнер &mdash; запущенный экземляр образа.

Dockerfile &mdash; файл с инструкциями для сборки образа и запуска контейнера.

OverlayFS &mdash; файловая система Linux. Объединяет несколько слоев в единую файловую систему. Каждый раз, когда происходят изменения в файле, OverlayFS создает новый слой.

|Образ|Контейнер|
|-----|---------|
|Потребляет только ресурсы файловой системы|Потребляет ресурсы процессора и оперативной памяти|
|Имеет имя и тег|Имеет имя, но не имеет тег|
|Состоит из слоев только для чтения|Содержит перезаписываемый слой для изменяемых файлов|

### Bind mounts, volumes и tmpfs
При использовании bind mounts передается существующая директория.
```bash
docker run -d \
  --name nginx-container \
  --mount type=bind,source="$(pwd)"/target,target=/app \
  nginx:latest
```
Volumes стоит использовать при создании директории для хранения персистентных данных &mdash; инфорации, которая сохраняет свое состояние и доступа после завершения программы или перезагрузки системы. 
```bash
docker run -d \
  --name nginx-container \
  --mount source=appdata,target=/app \
  nginx:latest
```
Здесь ```type``` можно не указывать, так как по умолчанию он имеет значение ```volume```, что означает монтирование тома.

Создаёт и удаляет volumes Docker Daemon, а для bind mounts созданием и удалением директорий вы занимаетесь самостоятельно.
При использовании драйвера томов по умолчанию ```local``` содержимое тома сохраняется на файловой системе хоста по пути ```/var/lib/docker/volumes/<имя_тома>/_data/```.

Tmpfs mounts отличается тем, что данные хранятся в оперативной памяти, подходит для быстрого доступа к временным маленьким файлам.

## Структура Dockerfile
Сборка образа выполняется при помощи команды ```docker build```
```bash
docker image build [OPTIONS] PATH | URL | -
```
где ```OPTIONS``` &mdash; необязательные опции и ```PATH | URL | -``` &mdash; контекст сборки образа. Чаще всего используется ```PATH``` для локального пути, можно указать ```URL```, например, для Git-репозитория или ```-``` для stdin. Если явно не указывать, то команда ```docker build``` будет пытаться найти Dockerfile в данной директории контекста.
### Важное замечание
Рассмотрим пример:
```bash
# tree .
├── app1
│   ├── Dockerfile
│   └── run.py
├── app2
│   ├── Dockerfile
│   └── run.py
└── Dockerfile.custom
```
Если запустить команду ```docker build``` с указанием папки app1 или app2, то файлы из других папок не будут доступны. Но если запустить Dockerfile.custom, лежащий в общей директории, то будут доступны все файлы. Используется опция ```-f <путь к Dockerfile>```.

### Часто используемые инструкции Dockerfile
- ```FROM``` Указывает базовый образ 

- ```RUN``` Выполняет команды сборки контейнера 

- ```COPY``` Копирует файлы в контейнер 

- ```ADD``` Добавляет локальные или удаленные файлы и каталоги 

- ```ENTRYPOINT``` Указывает исполняемую команду 

- ```CMD``` Указывает команды "аргументы" 

- ```WORKDIR``` Определяет рабочую директорию 

- ```ENV``` Определяет переменные окружения 

- ```ARG``` Определяет переменные на время сборки 

Для создания собственного базового образа используется ```FROM scratch```.

Если ```ENTYPOINT``` не задан, то командой запуска будет ```/bin/sh/ -c```. Рекомендуется указание абсолютных путей.

Переопределить параметры ```CMD``` можно двумя способами:
```bash
docker run --command run2.py my-container
или
docker run my-container run2.py
```

Инструкция ```WORKDIR``` может создать каталог, если его не существует.

Опция ```-t``` указывет tag, по умолчанию будет ```latest```.

## Практика: создайте собственный контейнер
Запуск контейнера с базой данных PostgreSQL в фоновом режиме
```bash
docker run --name postgres-db \
-e POSTGRES_PASSWORD=apipass \
-e POSTGRES_DB=api \
-e POSTGRES_USER=apiuser \
-p 5432:5432 \
-d postgres:16.2-alpine
```
Веб-приложение на FastAPI &mdash; современный фреймворк для создания API на Python, где фреймворк &mdash; каркас для разработки приложений &mdash;, которое подключается к базе данных PostgreSQL и возвращает информацию о ней.
```python
from fastapi import FastAPI
import psycopg2
import os

app = FastAPI()

# Чтение параметров переменных окружения. Если переменная не установлена, то берется второй аргумент.
DATABASE_HOST = os.getenv("API_DB_HOST", "localhost")
DATABASE_PORT = os.getenv("API_DB_PORT", "5432")
DATABASE_NAME = os.getenv("API_DB_NAME", "api")
DATABASE_USER = os.getenv("API_DB_USER", "apiuser")
DATABASE_PASS = os.getenv("API_DB_PASS", "apipass")

# Подключение в формате: postgresql://пользователь:пароль@хост:порт/база_данных
DATABASE_URL = f"postgresql://{DATABASE_USER}:{DATABASE_PASS}@{DATABASE_HOST}:{DATABASE_PORT}/{DATABASE_NAME}"

conn = psycopg2.connect(DATABASE_URL)
cursor = conn.cursor()

@app.get("/")
async def root():
    cursor.execute(
        "SELECT version();"
    )
    item = cursor.fetchone()
    return {"message": "Hello World",
            "postgres_version": item[0]}

@app.get("/hello/{name}")
async def say_hello(name: str):
    return {"message": f"Hello {name}"}
```
Создать файл ~/simple_python_app/requirements.dev.txt со следущим содержимым:
```python
fastapi==0.110.2
uvicorn[standard]==0.29.0
psycopg2-binary==2.9.9
```
И создать файл ~/simple_python_app/requirements.txt:
```python
fastapi==0.110.2
uvicorn[standard]==0.29.0
psycopg2==2.9.9
```
где ```uvicorn``` &mdash; веб-сервер для Python.
Создаем виртуальное окружение:
```bash
python3 -m venv venv
```
где ```-m``` &mdash; специальный флаг, запуск скрипта как модуля (встроенную библиотеку), ```venv```(первый) &mdash; название модуля, ```venv```(второй) &mdash; название папки, в которой будет создано окружение.
Активация виртуального окружения:
```bash
source ./venv/bin/activate
```
Установка зависимостей:
```bash
pip install -r requirements.dev.txt
```
```
Запускаем приложение:
```bash
uvicorn app:app --host 0.0.0.0 --port 8000
```
Теперь в браузере можем увидеть наше веб-приложение, перейдя по ```http://localhost:8000/```.
![веб-приложение](./images/screenshots/localhost8000.png)
Кроме того, можно протестировать с другого устройства в той же сети. Узнаем внутренний IP компьютера:
```bash
hostname -I | awk '{print $1}'
```
Вводим в браузере на другом устройстве ```http://<внутреннийIP>:8000/```
![веб-приложение_с_телефона](./images/screenshots/192168019.png)
Остановите приложение, нажав Ctrl + C.

