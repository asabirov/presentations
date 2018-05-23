Как устроен CI/CD в Apliteni
===

--- 

# Способы доставки приложений

* Ansible на bare metal
* Helm/kubectl в кластер Kubernetes

---

# Ansible

![](https://www.tutorialspoint.com/ansible/images/ansible_works.jpg)

---

# Когда выкатываем на bare metal

Высокий IO на диск


---

# Kubernetes
![](https://www.redhat.com/cms/managed-files/kubernetes-diagram-2-824x437.png)


---

# Kubernetes

![](https://www.jorgedelacruz.es/wp-content/uploads/2018/01/kubernetes-021_ccb09431d5a08b130c35f1a875e724d4.png)


---

# Когда выкатываем в Kubernetes

Во всех остальных случаях

---

# kubectl


```shell
$ kubectl apply -f app/ --namespace=app
pod "app" created
service "app" created
ingress "app" created
```

---

# Спецификации kubernetes

```
app/
    deployment.yaml
    service.yaml
    ingress.yaml
```
---

# deployment.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: appx-deployment
spec:
  replicas: 2
  selector:
    template:
      metadata:   
      	labels:
          app: appx
    spec:
      containers:
      - name: appx
        image: apliteni/appx
        ports:
        - containerPort: 80      	   
```

---
# kind
```yaml
kind: Deployment
...
```
---

# Варианты kind (объекты kubernetes)
Рабочие объекты: Pod, StatefulSet, Job, CronJob, ...
Discovery и балансировка: Service, Ingress, ...
Конфиги: ConfigMap, Secret, ...
Хранилище: StorageClass, PersistentVolumeClaim, Volume, ...

---

# apiVersion
```yaml
...
apiVersion: apps/v1
...
```

---

# spec

```yaml
...
spec:
  replicas: 2 # количество реплик
  selector:
    template:
      metadata:   
      	labels:
          app: appx   # ставит метку 'app' со значением 'appx'
    spec:
      containers:
      - name: appx     # запустить docker-контейнер 
        image: apliteni/appx      # образ с dockerhub
        ports:                    
        - containerPort: 80     
```

---
# Helm-Tiller
![](https://api.monosnap.com/rpc/file/download?id=FktkYR3OhKCVCwQwBchNM1gpHxMLiL)

---

# Helm
* gotemplates
* ```helm status RELEASE```
* ```helm dep```
* ```helm test RELEASE```
* ```helm rollback```
* Наличие готовых chart'ов

---
# helm
```
helm install app-chart/ --namespace=app --name=app-service-1
```
---
# Chart

```
app-chart/
    Chart.yaml
    values.yaml    
    requirements.yaml
    templates/
         deployment.yaml
         service.yaml
         ingress.yaml
```

---

# Chart.yaml

```yaml
apiVersion: v1
version: 0.1.1  
description: Management system for GDBC
name: gdbc-management
```

---
# values.yaml
```
resources:
  requests:
    memory: 512Mi
    cpu: 300m
    storage: 1Gi

image:
  repository: registry.domain.com:9999
  name: apliteni/gdbc/gdbc-management
  tag: ""
  pullPolicy: IfNotPresent

service:
  port: 80
  logLevel: DEBUG
```
---

# templates/deployment.yaml

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
# templates/_helpers.tpl

```
{{- define "project.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" -}}
{{- end -}}
```


---

# CI/DI


---


# Деплой со стороны разработчика

```bash
git push origin master
```

---

# CI/DI Pipeline

![](https://api.monosnap.com/rpc/file/download?id=NSQw93RpEBWjTHVCyb7lp06zxdx9PW)


---

# Подготовка проекта

* Dockerfile (Dockerfile.pre_build, Dockerfile.final_build)
* helm/
* .gitlab-ci.yml

---	

# Контейнеризация 
* Сборка в 2 стадии: pre_build, final_build
* Один итоговый образ
* Внешняя конфигурация параметры окружения (ENV)
* Версионирование приложения и образа

---

# Почему 2 стадии сборки образа?

* Тестам нужны исходники 
* Тестам нужны jest, cypress, runner'ы, go-cmd и т.д.
* Финальному образу минимальный размер

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

# Параметры окружения (ENV)

```yaml
apiVersion: apps/v1
kind: Deployment
...
spec:
  template:
    spec:
      containers:
        - name: appx
          env
            - name: LOG_LEVEL
              value: {{ .Values.service.logLevel }}
```              
---

# На стороне приложения
```go
import "os"

func getLogLevel() bool {
	return os.Getenv("LOG_LEVEL")
}
```

---

# Версионирование

* COMMIT SHA
* version.go, version.ts

--- 

# helm/

```
helm/
    Chart.yaml
    requirements.yaml
    values.yaml    
    values.dev.yaml    
    values.staging.yaml    
    templates/
         deployment.yaml
         service.yaml
         ingress.yaml
```
---
# .gitlab-ci.yml

```
...
stages:
  - pre_build
  - test
  - build_image
  - deploy
  - test_release
...
```

---

# Gitlab Registry
```yaml
before_script:
 - docker login -u gitlab-ci-token -p ${CI_JOB_TOKEN} 
     ${CI_REGISTRY}
```
---

# Cтадия pre-build
```yaml

build:
  stage: build
  script:       
    - export APP_VERSION=app-${CI_COMMIT_SHA}
    - docker build --pull
      --build-arg APP_VERSION=${APP_VERSION}   ←--       
      -t ${PRE_BUILD_IMAGE}
      -f Dockerfile.pre_build          ←--       
      .
      
    - docker push ${PRE_BUILD_IMAGE}

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
# Использование services (DinD)


```yaml
test:
  stage: test
  tags:
    - docker  ←--
  services:  
    - postgres:9.8   ←--
    - redis:2.5      ←--
  script: 
    - ping postgres
```

Docker executor не кэширует образы 
https://gitlab.com/gitlab-org/gitlab-ce/issues/17861
 


---

# Стадия build

```yaml
build:
  stage: build
  script:
    # Запуск prebuild-образа в контейнере
    - docker container create --name extract ${PRE_BUILD_IMAGE}
    
    # Вытягиваем бинарник из prebuild-образа
    - docker container cp extract:${PROJECT_PATH}/server ./server
    - docker container rm -f extract

    # Собираем новый образ 
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

# Проблема с helm install --upgrade

Есть баг с ```helm upgrade --install``` если релиз в статусе ```FAILED```
https://github.com/kubernetes/helm/issues/3353

---
# Стадия test release

```yaml
test_release:
  stage: test_release
  image: hypnoglow/kubernetes-helm
  tags:
    - docker
  only:
    - master
  environment:
    name: staging
  script:
    - helm init --client-only
     
    # Удалит старый pod с тестом, если он есть
    - ! kubectl delete pod ${CI_PROJECT_NAME}-test --namespace=${NAMESPACE} || true 
    
    # Запуск pod'а с тестом
    - helm test ${CI_PROJECT_NAME} 
```

---
# templates/tests/test-app.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ template "project.fullname" . }}-test"
  labels:
    heritage: {{ .Release.Service }}
    release: {{ .Release.Name }}
    chart: {{ .Chart.Name }}-{{ .Chart.Version }}
    app: {{ template "project.name" . }}
  annotations:
    "helm.sh/hook": test-success
spec:
  containers:
    - name: curl
      image: radial/busyboxplus:curl
      command: ["/bin/sh","-c"] 
      args: ['curl -s "{{ template "project.fullname" . }}:{{ .Values.service.port }}/_version" | grep {{ .Values.image.tag }} || incorrect_version_{{ .Values.image.tag }}']
  restartPolicy: Never
```



---
# Дальнейшее развитие
* Интеграционные тесты в kubernetes (helm + Job): 
```helm install --namespace=test-app-namespace ...```
* Автоматическое удаление старых образов
* Собственный репозиторий chart'ов


---

# Почитать
* https://docs.docker.com/engine/reference/builder/
* https://kubernetes.io/docs/home/
* https://docs.helm.sh/developers/
* https://docs.gitlab.com/ee/ci/yaml/

# Посмотреть
* https://www.youtube.com/playlist?list=PL8QZN5SAvhBMb4aNj4GZ7-b3efWkuTYYG