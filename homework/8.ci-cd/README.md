# Домашнее задание для к уроку 8 - CI/CD

> В рамках данного задания нужно продолжить работу с CI приложения из лекции.
> То есть для начала выполнения этого домашнего задания необходимо проделать то,
> что показывалось в лекции.
> Все задание должно выполняться применительно к файлам в директории practice/8.ci-cd/app

Переделайте шаг деплоя в CI/CD, который демонстрировался на лекции
таким образом, чтобы при каждом прогоне шага deploy в кластер применялись
манифесты приложения. При этом версия докер образа в деплойменте при апплае
должна подменяться на ту, что была собрана в шаге build.

Для этого самым очевидным способом было бы воспользоваться утилитой sed.

* Измените образ в деплойменте приложения (файл kube/deployment.yaml) на плейсхолдер.

Вот это

```yaml
image: nginx:1.12 # это просто плэйсхолдер
```

На это

```yaml
image: __IMAGE__
```

Файл deployment.yaml 
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: geekbrains
spec:
  progressDeadlineSeconds: 300
  replicas: 2
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
        - name: app
          image: __IMAGE__
          env:
            - name: DB_HOST
              value: database
            - name: DB_PORT
              value: "5432"
            - name: DB_USER
              value: app
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  key: db-password
                  name: app
            - name: DB_NAME
              value: users
          resources:
            limits:
              memory: "128Mi"
              cpu: "100m"
          ports:
            - containerPort: 8000
```

* Измените шаг деплоя в .gitlab-ci.yml,
чтобы изменять __IMAGE__ на реальное имя образа и тег

Это

```yaml
- kubectl set image deployment/$CI_PROJECT_NAME *=$CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG.$CI_PIPELINE_ID --namespace $CI_ENVIRONMENT_NAME
```

На это

```yaml
- sed -i "s,__IMAGE__,$CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG.$CI_PIPELINE_ID,g" kube/deployment.yaml
- kubectl apply -f kube/ --namespace $CI_ENVIRONMENT_NAME
```
```yaml
variables:
  K8S_API_URL: https://kubernetes.default

stages:
  - test
  - build
  - deploy

test:
  stage: test
  image: golang:1.14
  script:
    - echo OK

build:
  stage: build
  image: docker:19.03.12
  services:
    - docker:19.03.12-dind
  variables:
    DOCKER_DRIVER: overlay
    DOCKER_HOST: tcp://docker:2375 
    DOCKER_TLS_CERTDIR: ""
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build . -t $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG.$CI_PIPELINE_ID
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG.$CI_PIPELINE_ID

.deploy: &deploy
  stage: deploy
  image: bitnami/kubectl:1.16
  before_script:
    - export KUBECONFIG=/tmp/.kubeconfig
    - kubectl config set-cluster k8s --insecure-skip-tls-verify=true --server=$K8S_API_URL
    - kubectl config set-credentials ci --token=$(echo $K8S_CI_TOKEN | base64 --decode)
    - kubectl config set-context ci --cluster=k8s --user=ci
    - kubectl config use-context ci
  script:
    - sed -i "s,__IMAGE__,$CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG.$CI_PIPELINE_ID,g" kube/deployment.yaml
    - kubectl apply -f kube/ --namespace $CI_ENVIRONMENT_NAME
    - kubectl rollout status deployment/$CI_PROJECT_NAME --namespace $CI_ENVIRONMENT_NAME || (kubectl rollout undo deployment/$CI_PROJECT_NAME --namespace $CI_ENVIRONMENT_NAME && exit 1)

deploy:stage:
  <<: *deploy
  environment:
    name: stage
  variables:
    K8S_CI_TOKEN: $K8S_STAGE_CI_TOKEN
  only:
    - master

deploy:prod:
  <<: *deploy
  environment:
    name: prod
  variables:
    K8S_CI_TOKEN: $K8S_PROD_CI_TOKEN
  only:
    - master
  when: manual
  ```
  
> Вторую строчку шага деплоя (которая отслеживает статус деплоя) оставьте без изменений.

* Попробуйте закоммитить свои изменения, запушить их в репозиторий
(тот же, который вы создавали во время лекции на Gitlab.com)
и посмотреть на выполнение CI в интерфейсе Gitlab.

> Так как окружений у нас два (stage и prod), то помимо образа при апплае из CI
> нам также было бы хорошо подменять host в ingress.yaml.
> Попробуйте реализовать это по аналогии, подставляя в ингресс вместо
> плэйсхолдера значение переменной $CI_ENVIRONMENT_NAME
<img width="893" alt="Снимок экрана 2023-05-29 в 23 31 20" src="https://github.com/viktoriia-abalymova/geekbrains-conteinerization/assets/72358498/8ccee108-3321-4f36-a79c-41d3b880f116">

* Так же попробуйте протестировать откат на предыдущую версию,
при возникновении ошибки при деплое

Для этого можно изменить значение переменной DB_HOST в deployment.yaml на какое нибудь несуществующее.
Тогда при старте приложения оно не сможет найти БД и будет постоянно рестрартовать. CI должен в течении progressDeadlineSeconds: 300 и по.сле этого запустить процедуру отката.
При этом не должно возникать недоступности приложения, так как старая реплика должна продолжать работать, пока новая пытается стартануть.

