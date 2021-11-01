# Развёртывание в Kubernetes из GitLab

![Развёртывание в Kubernetes из GitLab](img/cover.png)

Это продолжение [предыдущего туториала про командную разработку с использованием GitLab](https://habr.com/ru/post/549862/). Фокус предыдущей статьи был на организации непрерывной поставки в работе команды. В этой статье мы уделим основное внимание именно практическим действиям необходимым для развёртывания из GitLab в Kubernetes.

А именно мы возьмём максимально простое но достаточно содержательное приложение на React.js, докеризуем его, затем развернём в Kubernetes локально при помощи Docker Desktop. После этого развернём его уже на Google Cloud Platform (GCP), и завершим разработкой CI/CD конвейера в GitLab для публикации нашего приложения в Google Kubernetes Engine.

Желательны но необязательны базовые знания
- Docker;
- Kubernetes;
- Git;
- Node.js;
- React;
- Bash.

В дальнейшем мы сделаем следующее.
- 🧱 Познакомимся c нашим приложением, обсудим из чего оно состоит.
- 🐳 Докеризуем наше приложение.
- ☸️ Развернём наше приложение в Kubernetes локально на Docker Desktop.
- ☁️ Обсудим особенности GCP и как нужно изменить наше приложение, а затем ещё раз развернём наше приложение в Kubernetes но уже в GCP.
- 🦊 Завершим наш туториал созданием конвейера для развертывания приложения в GCP при помощи GitLab.

![Разные этапы от докеризации до Kubernetes на Google Cloud Platform](img/gitlab-kubernetes.gif)

Время от времени я буду просить вас что-то сделать. Такие моменты помечены значком 🛠️. Пожалуйста выполняйте действия по мере чтения текста чтобы получить от данного туториала наибольшую пользу.

⚙️ Для того чтобы пройти этот туториал нужно чтобы эти программы были установлены на вашем компьютере.
- Bash Shell
- Git
- Node.js
- Docker Desktop
- Современный веб-браузер

Команды в тексте рассчитаны в `bash`, некоторые из них используют клиент `git` для командной строки. Если вы используете Windows, то самый простой способ получить и то и другое - установить [Git for Windows](https://git-scm.com/downloads).

Далее будем полагать что директории с исполняемыми файлами `node`, `git`, `docker` и `kubectl` включены в `PATH`.

## 🧱 Знакомство с приложением

📢 Знакомство с приложением будет включать эти шаги.
- Клонируем репозиторий и выясним из чего состоит наше приложение.
- Запустим приложение в консоли.
- Увидим каким приложение будет в дальнейшем в развёрнутом состоянии.

### Из чего состоит приложение

🛠️ Клонируйте репозиторий и перейдите в директорию приложения
```bash
git clone https://github.com/ntaranov/gitlab-kubernetes
cd gitlab-kubernetes
```

Приложение состоит из двух частей
- Single Page Application (**SPA**) - статический сайт на React который получает и изменяет данные, отправляя AJAX-запросы к сервису **API**.
- **API** - веб сервис, который предоставляет данные о "записавшихся" пользователях и может добавлять новые данные. API записывает данные, которые должны надёжно храниться, в файл. Мы используем файл чтобы избежать лишних для туториала сложностей, которые привнесла бы работа с базой данных. Дополнительно, файл позволяет нам поговорить о persistent volumes.

Зачем использовать именно такое приложение для этого туториала? Ну, это самое простое приложение, которое удовлетворяет таким требованиям. 
- Использует современный веб-фреймворк (React.js), требующий компиляции кода.
- Требует передачи параметра (`API_URL`) на этапе сборки.
- Содержит серверный код.
- Включает Front End и API для эмуляции микросервисной архитектуры.
- Имеет некоторое хранимое состояние (persistent state).
- Содержит тесты.

Я стремился сохранять приложение наиболее простым, поэтому я сознательно не включил такие вещи:
- аутентификацию и авторизацию;
- логи и телеметрию;
- обработку ошибок;
- взаимодейтсвие с базой данных;
- какого либо осмысленного покрытия тестами.

Цель туториала в том чтобы привести минимальные пример для развёртывания на Kubernetes и построения конвейера в GitLab. Даже несмотря на все упрощения туториал получился довольно огромный.

Внутри только что клонированного репозитория будут файлы и директории:
```javascript
api                 // Директория с файлами API
  /data/log         // Будем сюда сохранять данные
  /package.json     // NPM Манифест API
  /server.js        // Код API, не требует установки других пакетов
spa                 // Директория с файлами SPA
  /public/.         // Стандартные статические файлы React
  /src
    /components
      /Log.jsx      // Этот компонент шлёт запросы к API
    /App.js         // Главный компонент приложения SPA
    /App.test.js    // Тесты, у нас же должны быть тесты
    /config.js      // Загружает URL API из переменной окружения
    /index.js       // Корень React-приложения
  /package.json     // NPM Манифест SPA
  /package-lock.json  // С этими версиями пакетов работало
```

Остальные файлы либо стандартны для `create-react-app`, либо второстепенны для нашей задачи.

Обратите внимание, что компонент в файле `./spa/src/components/Log.jsx` посредством кода из `./spa/src/config.js` получает значение переменной окружения `REACT_APP_API_URL`. Если ничего не передать, код будет использовать значение по умолчанию годное для запуска на локальной машине, но если мы захотим развернуть код где-либо ещё, нам понадобится присвоить актуальное значение данной переменной окружения на этапе сборки, перед выполнением `npm run build`.

### Запускаем приложение локально

![Запускаем локально при помощи Node.js](img/local_node_800.png)

Запустите **API** и **SPA**. Проще всего это сделать открыв два консольных окна.

🛠️ В первом окне запустите **API**.
```bash
cd api
node server.js
```

🛠️ Во втором окне запустим **SPA**.
```bash
cd spa
```

1. Установите пакеты **npm**, указанные в `package-lock.json`.
    ```
    npm ci
    ```
2. **SPA** была создана при помощи `create-react-app`. Поэтому запустим его при помощи разработческого сервера из `react-scripts`.
    ```
    npm run start
    ```

🛠️ Откройте в браузере **SPA** по адресу 
```
http://localhost:3000/
```

🛠️ Нажмите кнопку **Log Me!**. Будет отправлен AJAX-запрос по адресу `http://localhost:3000/log` и данные User Agent текущего пользователя 
будут сохранены в файл `./data/log`. Новые данные будут отображены на странице под кнопкой.

Почему **API**, запущенный на порту `4000` получает запрос, направленный по адресу `http://localhost:3000/log`? Дело в том что разработческий сервер  `create-react-app` умеет работать как прокси, и перенаправляет запросы по адресу `http://localhost:4000/log`. Соответствующая настройка находится в `./spa/package.json`, обратите внимание на последнюю строчку `"proxy": "http://localhost:4000"`.

Запустить тесты для **SPA** вы можете командой
```
npm run test
```

Подготовить статический сайт для размещения на продуктиве можно командой
```
npm run build
``` 

### Что мы хотим видеть в конце?

Целевую архитектуру можно отобразить на этой картинке.

![Развёртывание в Kubernetes из GitLab](/img/gitlab_gke_800.png)

Для простоты мы храним данные в файле, но если перейти на хранение данных в БД, наша архитектура станет высокодоступной.

В первую очередь нам нужно упаковать наше приложение в качестве образа контейнера Docker.

## 🐳 Докеризуем наше приложение

![Докеризуем наше приложение](img/local_docker_800.png)

Хорошее введение в основы Docker [есть на официальном сайте](https://docs.docker.com/get-started/).

📢 Для того что бы докеризовать наше приложение мы вначале докеризуем **API** а затем **SPA**.

### Докеризуем API

Для того чтобы осуществить сборку контейнера Docker, [требуется создать файл с именем `Dockerfile`](https://docs.docker.com/get-started/02_our_app/). 

🛠️ Перейдите в директорию `./api` и создайте внутри файл `Dockerfile` с таким кодом.
```dockerfile
# используем Alpine Linux чтобы контейнер был меньше
FROM node:16-alpine
WORKDIR /app
RUN mkdir data
# копируем код сервиса
COPY server.js ./
CMD ["node", "server.js"]
```

🛠️ Убедитесь что Docker Desktop запущен. Запустите локальную сборку и присвойте образу тег `gitlab-course-api`
```bash
docker build . -t gitlab-course-api
```

🛠️ Запустите контейнер на основе образа
```bash
docker run --publish 4000:4000 -d gitlab-course-api
```
Эта команда выведет хэш контейнера, который вы сможете использовать чтобы остановить контейнер в дальнейшем.
Проверьте что сервис возвращает тестовую строку при запросе к `http://localhost:4000/log`.

### Докеризуем SPA

🛠️ Аналогично **API** добавим в директорию `./spa` `Dockerfile`. 
```dockerfile
# среда сборки
FROM node:14-alpine as builder
WORKDIR /app
ENV PATH /app/node_modules/.bin:$PATH

# копируем исходники React-приложение в среду сборки
COPY package.json ./
COPY package-lock.json ./
COPY src ./src
COPY public ./public

# устанавливаем NPM-пакеты в соответствии с package-lock.json
RUN npm ci

# мы должны передать этот параметр через --build-arg при запуске сборки
ARG API_URL
# осуществляем сборку приложение на React с использованием параметра
RUN REACT_APP_API_URL=$API_URL npm run build

# продуктивная среда
FROM nginx:stable-alpine
# копируем получившийся в результате сборки статический сайт из "тяжёлого" образа
# на маленький образ с веб-сервером nginx
COPY --from=builder /app/build /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

В данном случае `Dockerfile` вышел посложнее. Для того чтобы понять зачем такие сложности, давайте поймём как мы можем собирать **SPA**. Мы могли бы осуществлять сборку
- локально; 
- на отдельном сервере;
- прямо в контейнере.

В случае локальной сборки **SPA** среда сборки будет различаться от разработчика к разработчику.  
С последними двумя вариантами всё в порядке, для простоты выберем вариант сборки в контейнере.  
Мы хотели бы чтобы образ нашего приложения в продуктивной среде был как можно меньше и содержал минимум софта из соображений стабильности и безопасности. Так как наше приложение на React всего лишь статический сайт, на продуктиве нам достаточно лишь небольшого образа с веб-сервером. Однако, чтобы "собрать" наше приложение требуется **Node.js**, а также нужно установить все зависимости, а это целая куча пакетов. Чтобы разрешить это противоречие, мы и используем паттерн, который называется multi-stage build. Мы вначале собираем наше приложение в "тяжёлом" контейнере который по сути является средой сборки а затем копируем лишь получившийся статический сайт в легковесный образ, который и будет "раздавать" его в продуктивной среде.

🛠️ Соберите образ **SPA**,  запустив эту команду в директории со вновь созданным `Dockerfile`. Обратите внимание что мы передаём адрес **API** в качестве параметра чтобы в процессе сборки оказалась объявлена переменная `REACT_APP_API_URL`.
```bash
docker build .  --build-arg API_URL=http://localhost:4000 -t  gitlab-course-spa
```
Убедитесь что сборка была завершена успешно.

🛠️ Используйте следующую команду чтобы запустить контейнер на основе только что созданного образа.
```bash
docker run -it --publish 80:80 gitlab-course-spa
```

Если порт `80` по какой-то причине занят на вашем компьютере, вы можете опубликовать этот порт на другом порту хоста, например так `-p 8080:80`.

🛠️ Откройте в браузере страницу **SPA** `http://localhost`. Вы должны увидеть ту же страницу что и при локальном тесте, однако попытка вызвать **API** должна завершаться неуспешно с ошибкой связанной с [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS).

Это потому что с точки зрения same origin policy/CORS должны совпадать протокол, доменное имя и порт, а если они не совпадают, то это разные сайты. `http://localhost:80` и `http://localhost:4000` - разные сайты. Обычно для того, чтобы реализовать сценарий, похожий на наш, требуется чтобы веб-сервер **SPA** "разрешал" запросы к **API**, отправляя в браузер пользователя заголовки с нужным сайтом.

💡 CORS - одна из многих тем, типа настройки SSL/TLS, которые мы не будем включать этот туториал чтобы он не разросся на 100 страниц. На данном этапе нам было важно убедиться что оба веб-приложения работают. Когда мы развернём наше приложение в **Kubernetes**, мы сможем получать запросы к **SPA** и **API** на один домен и порт и различать их на основе `path`.

💡 Для локальной разработки часто используется утилита `docker-compose`. Мы могли бы использовать её чтобы реализовать взаимодействие **SPA** и **API**, а также их взаимодействие с внешним миром через вспомогательный reverse proxy. `docker-compose` тесно связан с оркестратором [**Docker Swarm**](https://docs.docker.com/engine/swarm/), продуктом **Docker**, и использует файлы конфигурации с совместимым синтаксисом. По той причине что в качестве оркестратора мы используем **Kubernetes**, мы будем выполнять наше приложение локально сразу в **Kubernetes**.

## ☸️ Развёртывание в Kubernetes

Мы уже научились запускать наше приложение в Docker, перейдём теперь к развёртыванию в Kubernetes.

![Развёртывание в Kubernetes](img/local_kubernetes_800.png)

Если вы еще не знакомы с Kubernetes, то я бы рекомендовал эти материалы.
- [What is Kubernetes?](https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/)
- Другие темы [Kubernetes Documentation -> Concepts -> Overview](https://kubernetes.io/docs/concepts/overview/)
- Туториал [Learn Kubernetes Basics](https://kubernetes.io/docs/tutorials/kubernetes-basics/).

Для знакомства с основами как правило достаточно дистрибуции Kubernetes, поставляемой вместе с Docker Desktop.

### Развернём приложение локально в Kubernetes на Docker Desktop

💡 Если у вас не получится создать все необходимые ресурсы из-за технических сложностей, вы можете либо заняться отладкой которая потребует основательно во всём разобраться либо перейти к следующему разделу где мы будем похожим образом разворачивать наше приложение в Kubernetes но на Google Kubernetes Engine (GKE). В другой среде те же технические проблемы могут не возникнуть. 

Создадим директорию, в которой мы будем хранить файлы ресурсов Kubernetes. Перейдите в корень репозитория и выполните команду.
```bash
mkdir kubernetes
```

Наше приложение развёрнутое в Kubernetes в Docker Desktop будет выглядеть так.

🛠️ Начнём с **SPA**. Добавим файл `./kubernetes/spa.yml` с таким кодом.

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: gitlab-course-spa
  namespace: gitlab-course
  labels:
    k8s-app: gitlab-course-spa
    project: gitlab-course
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: gitlab-course-spa
  template:
    metadata:
      name: gitlab-course-spa
      labels:
        k8s-app: gitlab-course-spa
    spec:
      containers:
      - name: gitlab-course-spa
        # так мы назвали наш docker image
        image: gitlab-course-spa
        imagePullPolicy: IfNotPresent
---
kind: Service
apiVersion: v1
metadata:
  name: gitlab-course-spa
  namespace: gitlab-course
  labels:
    k8s-app: gitlab-course-spa
    project: gitlab-course
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  selector:
    k8s-app: gitlab-course-spa
```

Данный файл содержит два YAML-документа разделённых `---`. 
- Первый документ определяет [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) для **SPA**. Deployment в данном случае содержит информацию о том какие Pods с какими контейнерами и в каком количестве мы хотим выполнять. Kubernetes полагается на `labels` для того чтобы понять какие Pods относятся к нашему Deployment. В данном случае в поле `selector` мы указываем что это Pods c `k8s-app: gitlab-course-spa`. Мы собираем образ локально, поэтому при `imagePullPolicy: IfNotPresent` в случае локального развёртывания будет использоваться локальная копия образа.
- Второй документ задаёт [Service](https://kubernetes.io/docs/concepts/services-networking/service/) для **SPA**. Service определяет каким образом наши Pods будут доступны как сетевой ресурс.

💡 Мы дополнительно определяем метку `project: gitlab-course` для всех ресурсов нашего проекта чтобы в дальнейшем иметь возможность получать их всех запросом по `label` вроде такого:
```bash
kubectl get all --selector=project=gitlab-course -n gitlab-course
``` 

🛠️ Кажется что у нас есть всё чтобы развернуть **SPA**, однако есть один нюанс. Дело в том, что мы обращались к **API** по адресу, заданному через `API_URL`, и это значение встроено в образ **SPA**. Пересоберём образ, используя значение `API_URL`, которое мы будем использовать в дальнейшем.
```bash
docker build ./spa --build-arg API_URL=http://spa.localtest.me/log -t gitlab-course-spa
```

🛠️ Создадим ресурсы **SPA** выполнив команду
```bash
kubectl apply -f kubernetes/spa.yml
```
Убедитесь, что в консоли подтверждается что оба ресурса созданы.

💡 Если вы захотите обновить образ Pod в Kubernetes, потребуется этот Pod пересоздать. Для **SPA** это можно сделать, например, этой командой.
```bash
kubectl rollout restart deployment gitlab-course-api -n gitlab-course
```

🛠️ Перейдём к **API**. Добавим файл `./kubernetes/api.yml` с таким кодом.
```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: gitlab-course-api
  namespace: gitlab-course
  labels:
    k8s-app: gitlab-course-api
    project: gitlab-course
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: gitlab-course-api
  template:
    metadata:
      name: gitlab-course-api
      labels:
        k8s-app: gitlab-course-api
    spec:
      containers:
      - name: gitlab-course-api
        # this is how we named our docker image
        image: gitlab-course-api
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - name: gitlab-course-pv
          mountPath: /app/data
      volumes:
      - name: gitlab-course-pv
        persistentVolumeClaim:
          claimName: gitlab-course-pv-claim
---
kind: Service
apiVersion: v1
metadata:
  name: gitlab-course-api
  namespace: gitlab-course
  labels:
    k8s-app: gitlab-course-api
    project: gitlab-course
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 4000
  selector:
    k8s-app: gitlab-course-api
```

Тут всё аналогично **SPA** кроме этой части:
```yaml
...
        volumeMounts:
        - name: gitlab-course-pv
          mountPath: /app/data
      volumes:
      - name: gitlab-course-pv
          persistentVolumeClaim:
            claimName: gitlab-course-pv-claim
...
```

Она связана с тем что мы используем Persistent Volumes для того чтобы данные, сохраняемые **API** могли жить дольше чем Pod. 
- В `volumes` мы объявляем новый диск и указываем что ресурсы для него должны быть получены с использованием Persistent Volume Claim `gitlab-course-pv-claim`.
- В `volumeMounts` мы монтируем этот диск в директорию `/app/data` где наше приложение хранит данные.

Однако для того чтобы это работало, нам требуется чтобы уже до этого был определён Persistent Volume Claim `gitlab-course-pv-claim`.

🛠️ Создайте новый файл `./kubernetes/volume.yml` с таким кодом.
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
reclaimPolicy: Delete
volumeBindingMode: Immediate
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: gitlab-course-pv
  labels:
    k8s-app: gitlab-course-api
    project: gitlab-course
spec:
  capacity:
    storage: 100Mi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    # данный пусть специфичен для Linux subsistem for Windows и транслируется в
    # C:\volumes\data\gitlab-course
    # эта директория должна быть создана перед созданием диска
    path: "/run/desktop/mnt/host/c/volumes/data/gitlab-course"
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/os
          operator: In
          values:
          - linux
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: gitlab-course-pv-claim
  labels:
    k8s-app: gitlab-course-api
    project: gitlab-course
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
  storageClassName: local-storage
  volumeName: gitlab-course-pv

```

Это, пожалуй, самый скучный момент туториала, но зато дальше так скучно уже не будет. Итак, разберёмся что тут и зачем.
Начнём с `PersistentVolumeClaim` с именем `gitlab-course-pv-claim`. Он определяет некий запрос на ресурсы и мы указываем его в определении **SPA**. Kubernetes расширяемая система, и потребности, в нашем случае `100MiB`, могут быть удовлетворены разным образом, например предоставлением облачного ресурса. Так как мы публикуем наше приложение локально, мы попросту создаём `PersistentVolume` с названием `gitlab-course-pv` чуть выше. Чтобы создать `PersistentVolume` требуется, в свою очередь, указать storage class который содержит информацию о том, какого рода диск нам требуется. Поэтому ещё выше мы создаём storage class с `storageClassName: local-storage`, который подразумевает что диск создан вручную.


🛠️ Примените приведённую выше конфигурацию.
```bash
kubectl apply -f kubernetes/volume.yml
```
Убедитесь что сообщение в консоли подтверждает что все три объекта созданы.

🛠️ Теперь у нас есть всё чтобы создать объекты **API**.
```bash
kubectl apply -f kubernetes/api.yml
```

Наши контейнеры уже выполняются в Kubernetes, но сервисы, которые мы создали доступны лишь из локальной сети внутри Kubernetes. Одним из способов предоставить доступ к нашему приложению извне является создание объекта который называется Ingress. Ingress определяет правила по которым наши сервисы будут доступны по HTTP. 

💡 Убедитесь что в вашей версии Kubernetes установлен ingress controller, например поддерживаемый сообществом `ingress-nginx`. Если вы не можете найти соответствующий Deployment, вам может понадобиться [установить его](https://kubernetes.github.io/ingress-nginx/deploy/). На момент написания этой статья команда для Docker Desktop выглядела так
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.0.4/deploy/static/provider/cloud/deploy.yaml
```

🛠️ Создадим файл `kubernetes/ingress.yml` с таким кодом. 
```yaml
kind: Ingress
apiVersion: networking.k8s.io/v1
metadata:
  name: gitlab-course-ingress
  namespace: gitlab-course
  labels:
    project: gitlab-course
spec:
  rules:
    - host: spa.localtest.me
      http:
        paths:
          - path: /log
            pathType: Prefix
            backend:
              service:
                name: gitlab-course-api
                port:
                  number: 80
  defaultBackend:
    service:
      name: gitlab-course-spa
      port:
        number: 80
```

В нашем случае в результате создания объекта Ingress будет создан reverse-proxy на основе nginx, который будет осуществлять передачу запросов к сервисам нашего приложения и ответов от них обратно клиенту. Суть данной конфигурации в том что мы принимаем запросы на порту 80 и перенаправляем все запросы на **SPA** помимо запросов по адресу `http://spa.localtest.me/log`, которые мы направляем к **API**. Мы тут используем `localtest.me` - доменное имя для которого возвращается IP `127.0.0.1` для всех поддоменов.

🛠️ Создайте ресурс Ingress.
```bash
kubectl apply -f kubernetes/ingress.yml
```

🛠️ Перейдите в браузере по адресу `http://spa.localtest.me`. Важно использовать именно этот домен потому для другого запрос к **API** по адресу `http://spa.localtest.me/log` будет заблокирован CORS.

👍 Поздравляю, вы развернули приложение в Kubernetes, хоть пока и локально. Как мы увидим в дальнейшем, использование другого провайдера не будет значительно сложнее.

## ☁️ Развернём наше приложение в GCP

![Развёртывание на GKE вручную](img/manual_gke_800.png)

### Создание проекта в GitLab

Мы будем использовать GitLab.com в качестве инсталляции GitLab чтобы избежать хлопот по установке и настройке локальной версии.

🛠️ Если у вас ещё нет учётной записи на GitLab.com, зайдите на https://gitlab.com и заведите её.

GitLab позволяет нам создать проект просто выполнив push в удалённый репозиторий.

🛠️ Воспользуемся для этого следующей командой:
```bash
GITLAB_USER_NAME=<user name>
git push https://gitlab.com/<user name>/gitlab-kubernetes
```
`<user name>` тут - ваше имя пользователя на GitLab.com. Дальше будем считать переменную `GITLAB_USER_NAME` в других скриптах имеющей то же значение.

Эта команда создаст приватный проект с именем `gitlab-kubernetes` внутри вашей учётной записи на GitLab.com.

GitLab предоставляет доступ к "переменным развёртывания", однако для этого наша ветвь должна быть "защищённой" (protected). Мы основательно разберёмся с этими переменными позже, но убедиться что наша ветвь `master` protected нам удобнее всего именно сейчас.

🛠️ При помощи браузера зайдите на страницу вашего проекта в GitLab `https://gitlab.com/<user name>/gitlab-kubernetes`;
- Выберите в меню слева **Settings** -> **Repository**;
- Разверните подменю **Protected branches**;
- Убедитесь что ветвь `master` перечислена в списке, и в колонке **Allowed to push** указаны хотя бы **Maintainers** (значение не равно **No one**).
- Если ветвь `master` в списке отсутствует, добавьте её (**Protect**) с вышеуказанными настройками.

### Создание кластера Kubernetes в GKE через GitLab

💡 Для выполнения задач данной части может понадобиться [создать "пробный" аккаунт](https://cloud.google.com/free). Вы можете получить $300 USD на "поиграть" с ресурсами на протяжении пробного периода и [дополнительно $200 USD](https://about.gitlab.com/partners/technology-partners/google-cloud-platform/) если воспользуетесь предложением GitLab. По крайней мере, таковы были предложения на момент написания этого туториала.

💡 "Из коробки" GitLab теперь также поддерживает интеграцию с Amazon EKS.

🛠️ Создайте проект с названием `gitlab-kubernetes` в Google Cloud Platform. Если вы не делали это раньше, то [тут написано как](https://cloud.google.com/resource-manager/docs/creating-managing-projects). 

💡 На самом деле можно вначале создать кластер в Kubernetes, а затем указать в GitLab ключ и токен нужного service account, но этот путь чуть длиннее и в большей степени провоцируют ошибки, поэтому мы "сжульничаем" и создадим кластер в Kubernetes сразу из GitLab, тогда все реквизиты будут заполнены автоматически.

🛠️ Создайте кластер Kubernetes с названием `gitlab-cluster-auto`. UI GitLab.com постоянно меняется, поэтому действия могут несколько отличаться, следуйте духу а не букве. В данный момент действия таковы:
- зайдите в проект `gitlab-kubernetes`;
- в левом меню выберите **Infrastructure** -> **Kubernetes clusters**;
- нажмите кнопку **Integrate with a cluster certificate** в центре страницы;
- справа выберите вкладку **Create new cluster** -> **Google GKE**.

Далее заполните форму.
- Укажите `gitlab-cluster-auto` в качестве названия кластера
- Убедитесь что выбран созданный для курса проект GCP `gitlab-kubernetes`
- Установите **Number of nodes** 1 - это дешевле и **API** не рассчитан на больше одной реплики.
- Можете выбрать **Machine type** поменьше, например `n1-standard-1`.
- Оставьте установленной опцию **GitLab-managed cluster**.
- Убедитесь что установлена опция **Namespace per environment** - мы не будем использовать много сред, но полезно включить эту опцию потому что это ближе к "реальной жизни".

Нажмите кнопку **Create Kubernetes cluster**. Дождитесь завершения процесса, убедитесь что отображается сообщение **Kubernetes cluster was successfully created**.

💡 На данном этапе может понадобиться зайти в вашу учётную запись GCP и дать GitLab необходимые разрешения.

🛠️ Убедитесь что на странице **Kubernetes clusters** напротив нашего кластера `gitlab-cluster-auto` нет ошибок.

👍 Теперь у нас есть кластер Kubernetes который мы сможем использовать в дальнейшем.
### Развёртывание в Кubernetes на GCP "вручную"

📢 При работе с кластером Kubernetes в GCP будем выполнять консольные команды в [Google Cloud Console Shell](https://shell.cloud.google.com). Это позволит
- упростить настройку авторизации при доступе к кластеру;
- сохранить локальные настройки `kubectl` прежними.

🛠️ Откройте в браузере страницу [Google Cloud Shell](https://shell.cloud.google.com).

🛠️ В Google Cloud Shell установлен `kubectl`, но нам нужно "подключить" его к кластеру. Для этого нам понадобится указать PROJECT_ID. Выведем список проектов в нашем окружении.
```bash
gcloud projects list
```
Например, для меня PROJECT_ID был `gitlab-kubernetes-326101`. Присвойте это значение переменной PROJECT_ID
```bash
PROJECT_ID=<ваше значение>
```

Теперь мы можем сохранить данные для подключения `kubectl`.
```bash
gcloud container clusters get-credentials gitlab-cluster-auto --zone us-central1-a --project $PROJECT_ID
```
Убедитесь, что выдача была
```
Fetching cluster endpoint and auth data.
kubeconfig entry generated for gitlab-cluster-auto.
```

Создадим namespace `gitlab-course`
```bash
kubectl create namespace gitlab-course
```

Давайте для удобства установим переменную `GITLAB_USER_NAME` в Google Cloud Shell
```bash
GITLAB_USER_NAME=<user name>
```

📢 Начнём развёртывание нашего приложения в GKE.
1. Создадим Persistent Volume Claim для **API**. 
1. Развернём **API**.
   - Загрузим образы **API** в репозиторий GitLab.
   - Cоздадим Deployment и Service **API**.
1. Развернём Ingress, ему будет присвоен внешний IP адрес. IP адрес нужен чтобы указать адрес **API** при создании образа **SPA**.
   - Протестируем **API**
1. Развернём **SPA**.
   - Пересоздадим образ **SPA** с использованием полученного адреса и загрузим образ **SPA** в репозиторий GitLab.
   - Наконец создадим Deployment и Service для **SPA**. 
   - Добавим правило для **SPA** в Ingress, обновим Ingress. Протестируем **SPA** и приложение целиком.

<!-- TODO нарисовать на диаграме -->

#### Создадим Persistent Volume Claim для **API**

💡 При работе с "облачными" провайдерами обычно требуется лишь создать Persistent Volume Claim, а сам диск и реализующий его "облачный" ресурс создаются автоматически средствами провайдера.

🛠️ Удалите StorageClass и PersistentVolume в `kubernetes/volume.yml`. Удалите `storageClassName: local-storage` в определении PersistentVolumeClaim, это приведёт к использованию [стандартного диска в GKE](https://cloud.google.com/kubernetes-engine/docs/concepts/persistent-volumes). Также заменим значение `namespace` на токен `{{NAMESPACE}}`.
```yml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: gitlab-course-pv-claim
  namespace: {{NAMESPACE}}
  labels:
    k8s-app: gitlab-course-api
    project: gitlab-course
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
```

💡 Идея в том что `namespace` в зависимости от того как и куда мы развёртываем приложение может может быть различным. По этой причине мы будем заменять `{{ТОКЕНЫ}}` на реальные значения перед применением файлов конфигурации. Мы могли бы использовать что-то типа Helm для работы с шаблонами, но мы же не хотим усложнять туториал, не так ли?

🛠️ Загрузите этот файл в Google Cloud Shell. Кнопка "Upload" находится в меню командной строки в интерфейсе Cloud Shell. На момент написания этого текста, опция находилась в подменю "**⋮**". 
По той причине что мы токенизировали файл, нам нужно вначале заменить токен на реальное значение. 
Например, так
```bash
sed "s/{{NAMESPACE}}/gitlab-course/g" volume.yml > volume-replaced.yml
```
Затем примените полученную конфигурацию
```bash
kubectl apply -f volume-replaced.yml
```

Или можете осуществить подстановку и применение конфигурации одной строчкой.
```bash
sed "s/{{NAMESPACE}}/gitlab-course/g" volume.yml | kubectl apply -f -
```

Проверьте следующей командой что Persistent Volume Claim создан.
```bash
kubectl get persistentvolumeclaim --all-namespaces
```

#### Развёртывание API

🛠️ Отредактируйте `kubernetes/api.yml`.
Замените `namespace` и в Deployment в Service таким образом.
```yaml
...
  namespace: {{NAMESPACE}}
...
```
Поступим с `image` аналогично, мы будем передавать точные версии образа Docker для того чтобы дать Kubernetes знать что Pod следует обновить.
```yaml
...
        image: {{API_IMAGE}}
...
```

Мы теперь можем подставить другое название образа, но наш локальный образ контейнера не будет доступ для GKE. Нам нужно разместить наш образ в сервисе доступном для Kubernetes. Загрузим локальный образ в GitLab Repository.

🛠️ В **локальной** консоли наберите
```bash
docker login registry.gitlab.com/$GITLAB_USER_NAME/gitlab-kubernetes
```
и введите свои данные для входа в GitLab.

🛠️ Присвойте образу тэг с префиксом нашего репозитория в GitLab.
```bash
docker tag gitlab-course-api registry.gitlab.com/$GITLAB_USER_NAME/gitlab-kubernetes/api
```

🛠️ Теперь мы можем загрузить наш образ в удалённый репозиторий. Выполните команду
```bash
docker push registry.gitlab.com/$GITLAB_USER_NAME/gitlab-kubernetes/api
```
Дождитесь окончания загрузки и убедитесь что она прошла успешно.


🛠️ В **Google Cloud Shell** наберите
```bash
docker login registry.gitlab.com/$GITLAB_USER_NAME/gitlab-kubernetes
```

🛠️ Есть нюанс, заключающийся в том, что для того чтобы скачать образ из нашего приватного репозитория на GitLab.com нужно предоставить некоторый секрет. Для этого мы можем загрузить секрет, который был создан по пути `~/.docker/config.json` когда мы выполнили команду `docker login`. Выполните эту команду в **Google Cloud Shell**.
```bash
kubectl create secret generic regcred \
    --from-file=.dockerconfigjson=$HOME/.docker/config.json \
    --type=kubernetes.io/dockerconfigjson \
    --namespace=gitlab-course
```
Последняя часть головоломки с настройкой образа - добавить информацию о нашем секрете для доступа к репозиторию в `kubernetes/api.yml`. Разместите этот код в конце определения Deployment сразу после `volumes:` на одном уровне с `containers:`
```yaml
...
      imagePullSecrets:
      - name: regcred
...
```

В Google Cloud мы будем использовать Ingress по умолчанию для передачи траффика с внешнего IP на Pod в Kubernetes. Эта реализация Ingress "под капотом" использует External HTTP/S Load Balancer и требует чтобы все сервисы были доступны как `NodePort`.
Изменим тип сервиса в `kubernetes/api.yml`.
```yaml
...
spec:
  type: NodePort
...
```

В итоге внутри `kubernetes/api.yml` должен получиться такой код:
```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: gitlab-course-api
  namespace: {{NAMESPACE}}
  labels:
    k8s-app: gitlab-course-api
    project: gitlab-course
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: gitlab-course-api
  template:
    metadata:
      name: gitlab-course-api
      labels:
        k8s-app: gitlab-course-api
    spec:
      containers:
      - name: gitlab-course-api
        image: {{API_IMAGE}}
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - name: gitlab-course-pv
          mountPath: /app/data
      volumes:
      - name: gitlab-course-pv
        persistentVolumeClaim:
          claimName: gitlab-course-pv-claim
      imagePullSecrets:
      - name: regcred
---
kind: Service
apiVersion: v1
metadata:
  name: gitlab-course-api
  namespace: {{NAMESPACE}}
  labels:
    k8s-app: gitlab-course-api
    project: gitlab-course
spec:
  type: NodePort
  ports:
  - protocol: TCP
    port: 80
    targetPort: 4000
  selector:
    k8s-app: gitlab-course-api
```

Отлично! Теперь наш образ доступен для Kubernetes и мы можем создать Pod!

🛠️ Загрузите файл `kubernetes/api.yml` в Google Cloud Console и выполните команду.
```bash
sed -e 's/{{NAMESPACE}}/gitlab-course/g' -e "s~{{API_IMAGE}}~registry.gitlab.com/$GITLAB_USER_NAME/gitlab-kubernetes/api~g" api.yml | kubectl apply -f -
```

Убедитесь что ресурсы созданы успешно.

💡 Обратите внимание что мы используем `~` в качестве разделителя во втором выражении `sed` потому что в имени образа уже встречается `/`.

💡 Если вы захотите обновить образ Pod в Kubernetes, потребуется этот Pod пересоздать. Для **API** это можно сделать, например, этой командой.
```bash
kubectl rollout restart deployment gitlab-course-api -n gitlab-course
```
Затем потребуется выполнить `kubectl apply` ещё раз, указав sha256 явно, добавив к имени образа конструкцию `@sha256:<hash>`.

#### Развёртывание Ingress

💡 Мы воспользуемся [nip.io](https://nip.io/) для того чтобы избежать необходимости регистрировать домен. Проблема тут в том, что для того чтобы указать правильный хост для **API** в определении Ingress, нам нужно знать внешний IP адрес Ingress. Однако IP адрес будет присвоен Ingress только после того как мы создадим Ingress. Решение в том чтобы вначале создать, а затем изменить Ingress. 

🛠️ Создадим Ingress. Измените файл `kubernetes/ingress.yml`.

```yaml
kind: Ingress
apiVersion: networking.k8s.io/v1
metadata:
  name: gitlab-course-ingress
  namespace: {{NAMESPACE}}
  labels:
    project: gitlab-course
spec:
  defaultBackend:
    service:
      name: gitlab-course-api
      port:
        number: 80
```

💡 Чтобы создать Ingress нам нужно указать или хотя бы одно правило либо бэкенд по умолчанию. Мы временно указываем сервис **API** потому что он уже создан.

🛠️ Загрузите этот файл в Google Cloud Shell.
```bash
sed "s/{{NAMESPACE}}/gitlab-course/g" ingress.yml | kubectl apply -f -
```

🛠️ Давайте дождёмся присваивания внешнего IP нашему Ingress. Это удобно сделать выполните эту команду
```bash
kubectl get ingress --all-namespaces --watch
```
Прекратите выполнение команды при помощи `Ctrl + C` когда внешний IP будет присвоен. Присвойте его переменной `EXTERNAL_IP`
```bash
EXTERNAL_IP=<внешний IP полученный предыдущей командой>
```
Для простоты я буду называть сам адрес тоже `EXTERNAL_IP`. 

🛠️ Перейдите в браузере по адресу `http://EXTERNAL_IP.nip.io`. Например, в процессе тестирования этого курса, адрес для меня выглядел `http://34.102.175.33.nip.io/log`

Убедитесь, получен ответ с кодом `200 OK`.

Итак, мы почти закончили развёртывать наше приложение в GKE, осталось развернуть **SPA** и обновить Ingress.

#### Развёртывание SPA

🛠️ Помните про нюанс с **SPA**? Внутри образа встроен адрес **API**, заданный через `API_URL`. Хорошие новости в том что мы уже создали Ingress и поэтому знаем адрес, который можем использовать в качестве `API_URL`. Пересоберём образ **SPA** с использованием этого адреса. Выполните эту команду в локальной консоли.
```bash
docker build ./spa --build-arg API_URL="http://$EXTERNAL_IP.nip.io" -t registry.gitlab.com/$GITLAB_USER_NAME/gitlab-kubernetes/spa
docker push registry.gitlab.com/$GITLAB_USER_NAME/gitlab-kubernetes/spa
```

🛠️ Отредактируйте `kubernetes/spa.yml`. Это будут те же правки что и в `kubernetes/api.yml` до того. 

Замените `namespace` и в Deployment и в Service таким образом.
```yaml
...
  namespace: {{NAMESPACE}}
...
```

Поступим с `image` аналогично.
```yaml
...
        image: {{SPA_IMAGE}}
...
```

Изменим тип сервиса в `kubernetes/spa.yml` на `NodePort`.
```yaml
...
spec:
  type: NodePort
...
```

Разместите секрет для доступа в репозиторий GitLab в конце определения Deployment на одном уровне с `containers:`
```yaml
...
      imagePullSecrets:
      - name: regcred
...
```

В итоге должно получиться такое:
```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: gitlab-course-spa
  namespace: {{NAMESPACE}}
  labels:
    k8s-app: gitlab-course-spa
    project: gitlab-course
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: gitlab-course-spa
  template:
    metadata:
      name: gitlab-course-spa
      labels:
        k8s-app: gitlab-course-spa
    spec:
      containers:
      - name: gitlab-course-spa
        image: {{SPA_IMAGE}}
        imagePullPolicy: IfNotPresent
      imagePullSecrets:
      - name: regcred
---
kind: Service
apiVersion: v1
metadata:
  name: gitlab-course-spa
  namespace: {{NAMESPACE}}
  labels:
    k8s-app: gitlab-course-spa
    project: gitlab-course
spec:
  type: NodePort
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  selector:
    k8s-app: gitlab-course-spa
```

🛠️ Загрузите `kubernetes/spa.yml` в директорию `~/gitlab-course` Cloud Shell и выполните команду.
```bash
sed -e 's/{{NAMESPACE}}/gitlab-course/g' -e "s~{{SPA_IMAGE}}~registry.gitlab.com/$GITLAB_USER_NAME/gitlab-kubernetes/spa~g" spa.yml | kubectl apply -f -
```

🛠️ Осталось обновить Ingress - и мы закончили. Давайте сделаем это сейчас. Укажем **SPA** в качестве сервиса по умолчанию и будем направлять запросы к **API** если путь равен `/log`. Обратите внимание что мы используем Ingress по умолчанию в GKE, и потому указываем `pathType: ImplementationSpecific`. 
```yaml
kind: Ingress
apiVersion: networking.k8s.io/v1
metadata:
  name: gitlab-course-ingress
  namespace: {{NAMESPACE}}
  labels:
    project: gitlab-course
spec:
  rules:
    - host: {{EXTERNAL_IP}}.nip.io
      http:
        paths:
          - path: /log
            pathType: ImplementationSpecific
            backend:
              service:
                name: gitlab-course-api
                port:
                  number: 80
  defaultBackend:
    service:
      name: gitlab-course-spa
      port:
        number: 80
```

🛠️ Удалите файл `ingress.yml` в Google Cloud Shell, а затем заново загрузите файл `kubernetes/ingress.yml` в Google Cloud Shell и примените изменения командой.
```bash
sed -e 's/{{NAMESPACE}}/gitlab-course/g' -e "s/{{EXTERNAL_IP}}/$EXTERNAL_IP/g" ingress.yml | kubectl apply -f -
```

🛠️ Зайдите в браузере на страницу `EXTERNAL_IP.nip.io` и убедитесь что наше приложение работает как предполагается. Вам может понадобиться подождать некоторое время пока конфигурация Ingress обновится в GCP. 

👍 Поздравляем, вы развернули приложение в Kubernetes в Google Cloud Platform! Осталось только организовать развёртывание приложения в GitLab.

## 🦊 Развёртывание в Kubernetes при помощи GitLab 

![Развёртывание в Kubernetes при помощи GitLab](img/gitlab_gke_800.png)

Самое сложное позади, осталось разработать pipeline в GitLab. Итак, начнём.

💡 Мы не будем показывать здесь как опубликовать приложение при помощи AutoDevOps (красивое!) потому что AutoDevOps и всё что с ним связано меняется крайне часто, и, кроме того мало подходит для чего-либо сложнее `Hello World!`. Тем не менее, я рекомендую пройти [официальный туториал](https://docs.gitlab.com/ee/topics/autodevops/quick_start_guide.html) чтобы составить впечатление о встроенных возможностях GitLab и использовать их по необходимости в дальнейшем.

💡 Я не думаю что реально предоставить решение в стиле "один размер подходит всем" так как чем "продуктивнее" конвейер тем он сложнее и тем сильнее зависит от платформы на которой происходит развёртывание. По этой причине мы не будем стремиться создать высокооптимизированный конвейер, а вместо этого сосредоточимся на создании решения которое просто понять и которое демонстрирует разные возможности.

📢 План такой
- Удалим ресурсы, созданные на предыдущем этапе
- Зададим переменные окружения
- Реализуем сборку и тестирование
- Реализуем развёртывание в GKE

### Удалим ресурсы, созданные на предыдущем этапе

🛠️ Будем удалять ресурсы "сверху вниз". Выполните эти команды в Google Cloud Shell.
```yaml
kubectl delete ingress gitlab-course-ingress -n gitlab-course
kubectl delete service gitlab-course-api -n gitlab-course
kubectl delete service gitlab-course-spa -n gitlab-course
kubectl delete deployment gitlab-course-api -n gitlab-course
kubectl delete deployment gitlab-course-spa -n gitlab-course
kubectl delete pvc gitlab-course-pv-claim -n gitlab-course
kubectl delete secret regcred -n gitlab-course
```

### Задаём переменные окружения

🛠️ Итак, наш конвейер будет использовать переменную `EXTERNAL_IP`, которая будет содержать внешний IP нашего Ingress. 
Давайте зададим её значение внутри GitLab. 

- Откройте в браузере страницу нашего проекта в GitLab `gitlab-kubernetes`
- В левом меню в подменю **Settings** выберите **CI/CD**. 
- Раскройте подменю **Variables**.
- Нажмите кнопку **Add Variable**, в поле **Key** введите `EXTERNAL_IP`, в поле **Value** введите значение, полученное вами на предыдущем этапе. На самом деле нет гарантии что GCP присвоит вновь созданному Ingress тот же IP, что и недавно удалённому, но вероятность этого высока, а если этого не произойдём, мы сможем легко это исправить.
- Нажмите кнопку **Add Variable** внизу формы. 

Готово!

Если вы отлаживаете ваш конвейер, вы также можете захотеть добавить переменную `CI_DEBUG_TRACE` со значением `true`. Если вы это сделаете, в логе job будут отображены значения всех переменных и параметров.

💡 Всегда включенный `CI_DEBUG_TRACE` создаёт риски для безопасности.

### Реализуем сборку и тестирование

🛠️ Создайте файл `.gitlab-ci.yml` в корне проекта. Добавьте этот код
```yaml
image: docker:stable
services:
# we will build our images by running docker daemon inside a container
- docker:dind
variables:
  DOCKER_DRIVER: overlay2
  SPA_DOCKER_IMAGE: $CI_REGISTRY_IMAGE/spa
  SPA_DOCKER_BUILDER_IMAGE: $CI_REGISTRY_IMAGE/spa-build
  API_DOCKER_IMAGE: $CI_REGISTRY_IMAGE/api
```
`DOCKER_DRIVER: overlay2` - Согласно [документации](https://docs.gitlab.com/ee/ci/docker/using_docker_build.html), `overlay2` эффективнее `vfs`. На самом деле это значение по умолчанию для shared runners, но мы оставили эту строчку на случай если вы захотите использовать self-hosted runners.

```yaml
  SPA_DOCKER_IMAGE: $CI_REGISTRY_IMAGE/spa
  SPA_DOCKER_BUILDER_IMAGE: $CI_REGISTRY_IMAGE/spa-build
  API_DOCKER_IMAGE: $CI_REGISTRY_IMAGE/api
```
Эти переменные мы завели для нашего удобства. Это названия наших образов внутри GitLab registry. Почему у нас только одна переменная для **API**, но две для **SPA**?  

Для **API** мы просто упаковываем `server.js` внутрь небольшого образа с Node.js. 

Для **SPA**, как вы помните, мы делали двухэтапную сборку. 
- С одной стороны, мы хотим чтобы образ в продакшне был как можно меньше. 
- С другой стороны, в нашем CD-конвейере мы будем запускать автотесты, а для этого нужны установленные модули NPM, т.е. для этого нам нужен "большой" образ. 

Решение в том чтобы разделить нашу двухэтапную сборку на сборку 2-ух образов - "побольше" и "поменьше".

🛠️ Кстати, давайте это сделаем прямо сейчас.

- Создайте файл `Build.Dockerfile` и скопируйте в него первую часть кода из `Dockerfile`
  ```dockerfile
  # build environment
  FROM node:14-alpine as builder
  WORKDIR /app
  ENV PATH /app/node_modules/.bin:$PATH

  # copy React source codes into the build image
  COPY package.json ./
  COPY package-lock.json ./
  COPY src ./src
  COPY public ./public

  # install NPM packages according to package-lock.json
  RUN npm ci

  # this param needs to be supplied as --build-arg when building the image
  ARG API_URL
  # building the react application using the param
  RUN REACT_APP_API_URL=$API_URL npm run build
  ```
- А вот исходный `Dockerfile` мы сократим. Удалим из файла код приведённый выше и параметризуем название образа, из которого мы будем в итоге копировать файлы.
  ```dockerfile
  ARG build
  # build environment
  FROM $build as builder

  # production environment
  FROM nginx:stable-alpine
  # copy the static site build result from our "heavy" build container
  # with NPM packages installed to a slim nginx web server image
  COPY --from=builder /app/build /usr/share/nginx/html
  EXPOSE 80
  CMD ["nginx", "-g", "daemon off;"]
  ``` 
  Как видите, просто и логично.
- Добавим ещё один, третий файл `Test.Dockerfile`, в котором мы и будем проводить тесты
  ```dockerfile
  # test environment
  FROM spa-build:latest
  RUN npm run test
  ```
  Да, всего две строчки.

💡 В качестве альтернативы, мы могли бы повторно использовать "сборочный" образ, передав команду запуска тестов через командную строку.

Теперь всё готово к тому чтобы реализовать все этапы связанные со сборкой и тестированием.
🛠️ Добавьте в `.gitlab-ci.yml` довольно большой кусок кода. 
```yaml
stages:
- build
- test
- package

before_script:
  - docker login -u gitlab-ci-token -p $CI_JOB_TOKEN $CI_REGISTRY

build_spa_builder:
  stage: build
  script:
  - docker build -f ./spa/Build.Dockerfile --build-arg API_URL="http://$EXTERNAL_IP.nip.io" -t $SPA_DOCKER_BUILDER_IMAGE:$CI_COMMIT_SHORT_SHA ./spa
  - docker tag $SPA_DOCKER_BUILDER_IMAGE:$CI_COMMIT_SHORT_SHA  $SPA_DOCKER_BUILDER_IMAGE:latest
  - docker push $SPA_DOCKER_BUILDER_IMAGE:$CI_COMMIT_SHORT_SHA
  - docker push $SPA_DOCKER_BUILDER_IMAGE:latest

test_spa:
  stage: test
  script:
  - docker run $SPA_DOCKER_BUILDER_IMAGE:latest sh -c "CI=true npm test"
  dependencies:
  - build_spa_builder

build_spa:
  stage: package
  script:
  - docker build -f ./spa/Dockerfile --build-arg build=$SPA_DOCKER_BUILDER_IMAGE:latest -t $SPA_DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA ./spa
  - docker tag  $SPA_DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA $SPA_DOCKER_IMAGE:latest
  - docker push $SPA_DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA
  - docker push $SPA_DOCKER_IMAGE:latest
  dependencies:
  - build_spa_builder

build_api:
  stage: package
  script:
  - docker build -f ./api/Dockerfile -t $API_DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA ./api
  - docker tag $API_DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA $API_DOCKER_IMAGE:latest
  - docker push $API_DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA
  - docker push $API_DOCKER_IMAGE:latest
```
Данный код определяет три этапа `build`, `test` и `package`. 
Эти этапы в свою очередь содержат jobs. Давайте обсудим какой job что делает.

- **build**
  - **build_spa_builder** - создаём образ "сборщика"
- **test**
  - **test_spa** - тестируем **SPA**
- **package**
  - **build_spa** - копируем файлы **SPA** на продуктивный образ
  - **build_api** - копируем файлы **API**

Команды, указанные в `before_script` будут выполнены перед каждым job. Мы используем `docker login` чтобы авторизовать runner для доступа к GitLab Docker Registry с использованием данных доступных через переменные окружения.
```yaml
  before_script:
  - docker login -u gitlab-ci-token -p $CI_JOB_TOKEN $CI_REGISTRY
``` 

Мы тут используем некоторый переменные, которые начинаются на `CI_`, это так называемые [переменные GitLab CI/CD](https://docs.gitlab.com/ee/ci/variables/). Их значения предоставлены нам платформой.

Большинство jobs выше устроены аналогично за исключением **test_spa** который проще потому что в итоге мы не загружаем образ в репозиторий. Если вы хотите убедиться, что перед запуском некого job завершится предыдущий, вы можете указать предыдущий в `dependencies`.  Разберём команды на примере **build_spa_builder**.

```yaml
  script:
  - docker build -f ./spa/Build.Dockerfile --build-arg API_URL="http://$EXTERNAL_IP.nip.io" -t $SPA_DOCKER_BUILDER_IMAGE:$CI_COMMIT_SHORT_SHA ./spa
  - docker tag $SPA_DOCKER_BUILDER_IMAGE:$CI_COMMIT_SHORT_SHA  $SPA_DOCKER_BUILDER_IMAGE:latest
  - docker push $SPA_DOCKER_BUILDER_IMAGE:$CI_COMMIT_SHORT_SHA
  - docker push $SPA_DOCKER_BUILDER_IMAGE:latest
```
Тут мы создаём образ, затем тегируем его меткой состоящей из значения нашей переменной и короткого хэша текущего коммита в репозитории GitLab `CI_COMMIT_SHORT_SHA`. Затем мы загружаем образ в Docker-репозиторий GitLab. При создании образа **SPA** мы также используем переменную `EXTERNAL_IP` которую задали ранее.

Целей в том чтобы тэгировать образ хэшем коммита две. 
- Во-первых, мы поймём из какой версии кода собран образ. 
- Во-вторых, для образов, на которые будут ссылаться YAML-файлы ресурсов, Kubernetes сможет понять что образ изменился и что нужно обновить Pod.

🛠️ Итак, волшебный момент запуска нашего конвейера настал. Обновите код в репозитории GitLab следующей командой в локальной консоли. Добавьте весь код, который вы еще не добавили в индекс, закоммитьте и сделайте push.
```bash
git add -A
git commit -m "Commit all the code remaining to build images with GitLab"
git push
```

🛠️ Откройте в браузере страницу **CI/CD** -> **Pipelines** в GitLab и убедитесь что сборка и тестирование начались. Перейдите на страницу нашей конкретной сборки и дождитесь её успешного завершения. При этом полезно пооткрывать страницы разных jobs и понаблюдать за обновляющейся выдачей в консоли.

Отлично! Мы уже собираем все нужные нам образы прямо в GitLab.

### Реализуем развёртывание в GKE

🛠️ Давайте сделаем последний шаг - реализуем развёртывание в Kubernetes на GCP при помощи GitLab. Итак, добавьте дополнительный `stage` в `.gitlab-ci.yml`.
```yaml
stages:
- build
- test
- package
- deploy
```

Затем после шагов сборки добавьте такой код.
```yaml
deploy_spa:
  stage: deploy
  image: "registry.gitlab.com/gitlab-org/cluster-integration/auto-deploy-image"
  # нам нужно тут задать environment иначе GitLab не передаст значения переменных начинающихся с KUBE_
  environment: production
  before_script:
  - |
    kubectl create secret -n "$KUBE_NAMESPACE" \
    docker-registry regcred \
    --docker-server="$CI_REGISTRY" \
    --docker-username="${CI_DEPLOY_USER:-$CI_REGISTRY_USER}" \
    --docker-password="${CI_DEPLOY_PASSWORD:-$CI_REGISTRY_PASSWORD}" \
    --docker-email="$GITLAB_USER_EMAIL" \
    -o yaml --dry-run | kubectl replace -n "$KUBE_NAMESPACE" --force -f -
  script:
  # создаём диск
  - sed -e "s/{{NAMESPACE}}/$KUBE_NAMESPACE/g" kubernetes/volume.yml | kubectl apply -f -
  # развёртываем API
  - sed -e "s/{{NAMESPACE}}/$KUBE_NAMESPACE/g" -e "s~{{API_IMAGE}}~$API_DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA~g" kubernetes/api.yml | kubectl apply -f -
  # развёртываем SPA
  - sed -e "s/{{NAMESPACE}}/$KUBE_NAMESPACE/g" -e "s~{{SPA_IMAGE}}~$SPA_DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA~g" kubernetes/spa.yml | kubectl apply -f -
  # обновляем ingress
  - sed -e "s/{{NAMESPACE}}/$KUBE_NAMESPACE/g" -e "s/{{EXTERNAL_IP}}/$EXTERNAL_IP/g" kubernetes/ingress.yml | kubectl apply -f -
```

Тут внимания достоин тот факт что вместо `docker: stable` мы используем другой образ. В принципе, нам подойдёт любой образ, имеющий `kubectl`, поэтому для простоты мы используем образ `registry.gitlab.com/gitlab-org/cluster-integration/auto-deploy-image` который GitLab использует для развёртывание в Kubernetes в режиме AutoDevOps. 

Также мы [пользуемся фактом что наш кластер интегрирован с GitLab](https://docs.gitlab.com/ee/user/project/clusters/deploy_to_cluster.html#deployment-variables). По этой причине мы можем полагаться на переменные окружения, которые начинаются с `KUBE_`, они будут переданы в конвейер автоматически.

Мы переопределили `before_scripts`.
- Мы не хотим чтобы перед выполнением команд нашего `job` была попытка залогиниться в `docker`.
- Мы хотим дать нашим Pods возможность загружать образы из GitLab Docker Registry.

Привожу полный код `.gitlab-ci.yml` который должен был получиться.

```yaml
image: docker:stable
services:
- docker:dind
variables:
  DOCKER_DRIVER: overlay2
  SPA_DOCKER_IMAGE: $CI_REGISTRY_IMAGE/spa
  SPA_DOCKER_BUILDER_IMAGE: $CI_REGISTRY_IMAGE/spa-build
  API_DOCKER_IMAGE: $CI_REGISTRY_IMAGE/api

stages:
- build
- test
- package
- deploy

before_script:
- docker login -u gitlab-ci-token -p $CI_JOB_TOKEN $CI_REGISTRY

build_spa_builder:
  stage: build
  script:
  - docker build -f ./spa/Build.Dockerfile --build-arg API_URL="http://$EXTERNAL_IP.nip.io" -t $SPA_DOCKER_BUILDER_IMAGE:$CI_COMMIT_SHORT_SHA ./spa
  - docker tag $SPA_DOCKER_BUILDER_IMAGE:$CI_COMMIT_SHORT_SHA  $SPA_DOCKER_BUILDER_IMAGE:latest
  - docker push $SPA_DOCKER_BUILDER_IMAGE:$CI_COMMIT_SHORT_SHA
  - docker push $SPA_DOCKER_BUILDER_IMAGE:latest

test_spa:
  stage: test
  script:
  - docker run $SPA_DOCKER_BUILDER_IMAGE:latest sh -c "CI=true npm test"
  dependencies:
  - build_spa_builder

build_spa:
  stage: package
  script:
  - docker build -f ./spa/Dockerfile --build-arg build=$SPA_DOCKER_BUILDER_IMAGE:latest -t $SPA_DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA ./spa
  - docker tag  $SPA_DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA $SPA_DOCKER_IMAGE:latest
  - docker push $SPA_DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA
  - docker push $SPA_DOCKER_IMAGE:latest
  dependencies:
  - build_spa_builder

build_api:
  stage: package
  script:
  - docker build -f ./api/Dockerfile -t $API_DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA ./api
  - docker tag $API_DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA $API_DOCKER_IMAGE:latest
  - docker push $API_DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA
  - docker push $API_DOCKER_IMAGE:latest

deploy_spa:
  stage: deploy
  image: "registry.gitlab.com/gitlab-org/cluster-integration/auto-deploy-image"
  # нам нужно тут задать environment иначе GitLab не передаст значения переменных начинающихся с KUBE
  environment: production
  before_script:
    - |
    kubectl create secret -n "$KUBE_NAMESPACE" \
    docker-registry regcred \
    --docker-server="$CI_REGISTRY" \
    --docker-username="${CI_DEPLOY_USER:-$CI_REGISTRY_USER}" \
    --docker-password="${CI_DEPLOY_PASSWORD:-$CI_REGISTRY_PASSWORD}" \
    --docker-email="$GITLAB_USER_EMAIL" \
    -o yaml --dry-run | kubectl replace -n "$KUBE_NAMESPACE" --force -f -
  script:
  # создаём диск
  - sed -e "s/{{NAMESPACE}}/$KUBE_NAMESPACE/g" kubernetes/volume.yml | kubectl apply -f -
  # развёртываем API
  - sed -e "s/{{NAMESPACE}}/$KUBE_NAMESPACE/g" -e "s~{{API_IMAGE}}~$API_DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA~g" kubernetes/api.yml | kubectl apply -f -
  # развёртываем SPA
  - sed -e "s/{{NAMESPACE}}/$KUBE_NAMESPACE/g" -e "s~{{SPA_IMAGE}}~$SPA_DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA~g" kubernetes/spa.yml | kubectl apply -f -
  # обновляем ingress
  - sed -e "s/{{NAMESPACE}}/$KUBE_NAMESPACE/g" -e "s/{{EXTERNAL_IP}}/$EXTERNAL_IP/g" kubernetes/ingress.yml | kubectl apply -f -
```

🛠️ Закоммитьте изменения и выполните push в репозиторий
```bash
git add -A
git commit -m "Commit the code to deploy to GKE with GitLab"
git push
```

Перейдите по адресу `http://EXTERNAL_IP.nip.io` в браузере и убедитесь что приложение заработало. Может понадобиться подождать несколько минут чтобы все ресурсы успели обновиться. 

💡 Если приложение не открывается, но Deployments функционируют без ошибок, особенно если при переходе по адресу выше вы получаете код 404, есть вероятность что GCP присвоил вновь созданному Ingress другой внешний IP. Это легко исправить!
- Узнайте новый адрес IP
  ```bash
  kubectl get ingress -n <namespace созданнае GitLab>
  ```
- Измените значение переменной `EXTERNAL_IP` в GitLab соответственно.
- Перезапустите конвейер в GitLab.

## 🎉 Поздравляю! 

Вы докеризовали приложение на React.js, затем развернули его в Kubernetes вручную и в итоге создали конвейер непрерывного развёртывание этого приложения в Kubernetes при помощи GitLab!

🧹 Не забудьте удалить ненужные ресурсы, созданные на Google Cloud Platform.
