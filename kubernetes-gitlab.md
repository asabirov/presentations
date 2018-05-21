# Как работает Ansible

![](https://www.tutorialspoint.com/ansible/images/ansible_works.jpg)

---

# Как работает Kubernetes
![](https://harishnarayanan.org/images/writing/kubernetes-django/kubernetes-architecture.svg)

---


# Когда использовать Ansible
* Выделенный сервер
* Редкие деплои (1-2 раза в год) 

---

# Когда использовать Kubernetes
* Не важно на чем хостить
* Регулярные и автоматические деплои
* Автоматическое масштабирование
* Отказоустойчивость

----

# Подключение к кластеру K8s
```
kubectl config set-credentials cluster-admin --client-certificate=~/.kube/admin.crt --embed-certs=true
kubectl config set-cluster e2e --server=https://1.2.3.4
kubectl config set-context gce --user=cluster-admin
```
---


# Деплой в K8s


```
kubectl apply -f app/ --namespace=app
```

---
# Спеки

```
app/
    deployment.yaml
    service.yaml
    ingress.yaml
```

# Пример spec'а

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  minReadySeconds: 5
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```
---

# Плюсы kubectl
* Все делается одной утилитой ```kubectl delete```, ```apply```, ```create```
* Все инструменты для отладки ```kubectl describe```, ```logs```, ```get```


---


# Минусы kubectl

* Все приложения в одну кучу 
* Нет поддержки переменных в спеках
* Нельзя использовать сторонние спеки без доработки


---
# Деплой через Helm
```
helm install app-chart/ --namespace=app --name=app-release
```
---
# Helm Chart

```
app-chart/
    Chart.yaml
    values.yaml    
    templates/
         deployment.yaml
         service.yaml
         ingress.yaml
```

---

# Пример Chart.yaml

```yaml
apiVersion: v1
version: 0.1.0    # Helm chart version
description: GDBC
name: gdbc-management-api
```
---
# Пример values.yaml
```
resources:
  requests:
    memory: 512Mi
    cpu: 300m
    storage: 1Gi

image:
  repository: gitlab.x.apli.tech:1443
  name: apliteni/gdbc/gdbc-management-api
  tag: ""
  pullPolicy: IfNotPresent

service:
  port: 80
  logLevel: DEBUG
  debug: true
```
---

# Пример шаблона deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ template "project.fullname" . }}
  labels:
    name: {{ template "project.fullname" . }}
    ...
spec:
  replicas: {{ .Values.deploymnent.replicas }}
  selector:
    matchLabels:
      name: {{ template "project.fullname" . }}
  template:
    metadata:
      labels:
        name: {{ template "project.fullname" . }}
      containers:
        - name: {{ template "project.name" . }}
          image: "{{ .Values.image.repository }}/{{ .Values.image.name }}:{{ .Values.image.tag }}"
```

---

# Минусы Helm
* Нужно предустанавливать контроллер Tiller в каждый кластер
* Всё-равно нужно использовать kubectl для отладки
* Нельзя деплоить одно приложение в несколько namespace'ов


---

# Плюсы Helm
* Большой выбор готовых Чартов
* Вынос настроек из spec'ов  
* Удобная сводка состояний по имени релиза ```helm status RLS```
* Откат релиза ```helm rollback```
* Тестирование релиза ```helm test```

---

# Идеальный деплой

```bash
git push origin master
```
---

# Как его достичь?

* В каждом репозитории собственный helm chart.
* Инструкция для CI Pipeline'а

---	

# Структура проекта
```
app/
  helm/
    templates
    Chart.yaml
    values.yaml
  Dockerfile.pre_build
  Dockerfile.final_build
  .gitlab-ci.yaml  

  ...
  [файлы приложения]
```
---
# Докер образ
* Один проект - один образ
* Базовый образ alpine (всего 2мб)
* Multi-stage: сборка в первом образе, артефакты во втором
* Все настройки пробрасываются в ENV параметрах
* Версионирование (commit sha, VERSION файл или в исходнике)

--- 
# Helm Chart
* Spec'и для работы приложения: deployement, service, ingress, configmap и т.д.
* Настройки в ```values.yaml``` или по окружениям ```values.dev.yaml```, ```values.staging.yaml```.

---
# gitlab-ci.yml

Должны быть описаны все стадии выкатки: ```pre_build```, ```test```, ```build```, ```deploy```, ```test_release```

---


---
# Для чего pre_build?
Тестам нужны исходные коды тестов.

---


# Pre-build стадия
```yaml
before_script:
   # Авторизация в Gitlab Registry
   - if type docker > /dev/null; then
      docker login -u gitlab-ci-token -p ${CI_JOB_TOKEN} ${CI_REGISTRY};
    fi

build:
  stage: build
  script:       
    - docker build --pull
      -t ${PRE_BUILD_IMAGE}
      --build-arg APP_VERSION=${CI_COMMIT_SHA} # Версия
      --build-arg DATE=${DATE}
      --build-arg PROJECT_PATH=${PROJECT_PATH}
      .
      -f Dockerfile
    - docker push ${PRE_BUILD_IMAGE}

```
---
# Dockerfile.pre_build

```
FROM golang:1.9

ARG PROJECT_PATH
ARG APP_VERSION

RUN mkdir -p $PROJECT_PATH
WORKDIR $PROJECT_PATH

RUN go get -u github.com/golang/dep/...

COPY Gopkg.toml Gopkg.lock ./

RUN dep ensure -vendor-only

COPY ./ ./

RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo \
    		-ldflags "-X \"main.version=${APP_VERSION}\" -X \"main.buildAt=${DATE}\"" \
    		-o server .
```

---
# Стадия test

```yaml
test:
  stage: test
  script:
    - docker pull ${PRE_BUILD_IMAGE}
    - docker run -w ${PROJECT_PATH} ${REGISTRY_BUILD_IMAGE} 
        make test
```    
---    
# Особенность интеграционных тестов

```yaml
test:
  stage: test
  tags:
    - docker       # выбор docker-in-docker runner'а
  services:        # поднимет сервисы в контейнерах
    - postgres:9.3
    - redis:2.5
  script: 
    - ...
```

```
$ ping postgres
PING postgres (10.2.2.2): 56 data bytes
64 bytes from 10.2.2.2: icmp_seq=0 ttl=53 time=95.219 ms
```

---

# Стадия build

```yaml
build:
  stage: build
  script:
    - docker container create --name extract ${PRE_BUILD_IMAGE}
    
    # Вытягиваем бинарник
    - docker container cp extract:${PROJECT_PATH}/server ./server
    - docker container rm -f extract
    
    - docker build --no-cache
      -t ${CI_REGISTRY_IMAGE}:latest
      -t ${CI_REGISTRY_IMAGE}:${APP_VERSION}
      .
      -f Dockerfile.final_build
    - docker push ${CI_REGISTRY_IMAGE}
```     

---
# Стадия deploy

```yaml 

deploy:
  stage: deploy
  image: "${CI_REGISTRY}/infra/kubernetes-helm"
  tags:
    - docker
  only:
    - master
  environment:
    name: production       
  script:
    - helm init --client-only
    
    - export DEPLOYS=$(helm ls | grep ${CI_PROJECT_NAME} | wc -l)

    - if [ ${DEPLOYS}  -eq 0 ]; then
        helm install --name ${CI_PROJECT_NAME} --namespace=${NAMESPACE} --values helm/values.prod.yaml ./helm --set image.tag=${APP_VERSION};
      else
        helm upgrade ${CI_PROJECT_NAME} --namespace=${NAMESPACE} --values helm/values.prod.yaml ./helm --set image.tag=${APP_VERSION};
      fi
```
---

# Авторизация в кластере K8s

![](https://api.monosnap.com/rpc/file/download?id=hhAvIEZAdtKBuDBU9rdePmotvfOHes)

---
# Инъекция конфига кластера

```yaml 

deploy:
  ...
  environment:
    name: production        #    ←-- этого достаточно
```

# Стадия test_release 
TODO

```yaml

test_release:
  stage: test_release
  image: "${CI_REGISTRY}/infra/kubernetes-helm"
  tags:
    - docker
  only:
    - master
  environment:
    name: staging
  script:
    - helm init --client-only
    - helm status ${CI_PROJECT_NAME}
    - helm test ${CI_PROJECT_NAME} --cleanup
```

---
# Тест

./helm/templates/tests/test-app.yaml
```yaml
TODO
```