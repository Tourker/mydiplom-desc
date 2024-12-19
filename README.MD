# Дипломный практикум в Yandex.Cloud - Мирошниченко Никита FOPS-13
  * [Цели:](#цели)
  * [Этапы выполнения:](#этапы-выполнения)
     * [Создание облачной инфраструктуры](#создание-облачной-инфраструктуры)
     * [Создание Kubernetes кластера](#создание-kubernetes-кластера)
     * [Создание тестового приложения](#создание-тестового-приложения)
     * [Подготовка cистемы мониторинга и деплой приложения](#подготовка-cистемы-мониторинга-и-деплой-приложения)
     * [Установка и настройка CI/CD](#установка-и-настройка-cicd)
  * [Что необходимо для сдачи задания?](#что-необходимо-для-сдачи-задания)
  * [Как правильно задавать вопросы дипломному руководителю?](#как-правильно-задавать-вопросы-дипломному-руководителю)

**Перед началом работы над дипломным заданием изучите [Инструкция по экономии облачных ресурсов](https://github.com/netology-code/devops-materials/blob/master/cloudwork.MD).**

---
## Цели:

1. Подготовить облачную инфраструктуру на базе облачного провайдера Яндекс.Облако.
2. Запустить и сконфигурировать Kubernetes кластер.
3. Установить и настроить систему мониторинга.
4. Настроить и автоматизировать сборку тестового приложения с использованием Docker-контейнеров.
5. Настроить CI для автоматической сборки и тестирования.
6. Настроить CD для автоматического развёртывания приложения.

---
## Этапы выполнения:


### Создание облачной инфраструктуры

Для начала необходимо подготовить облачную инфраструктуру в ЯО при помощи [Terraform](https://www.terraform.io/).

Особенности выполнения:

- Бюджет купона ограничен, что следует иметь в виду при проектировании инфраструктуры и использовании ресурсов;
Для облачного k8s используйте региональный мастер(неотказоустойчивый). Для self-hosted k8s минимизируйте ресурсы ВМ и долю ЦПУ. В обоих вариантах используйте прерываемые ВМ для worker nodes.

Предварительная подготовка к установке и запуску Kubernetes кластера.

1. Создайте сервисный аккаунт, который будет в дальнейшем использоваться Terraform для работы с инфраструктурой с необходимыми и достаточными правами. Не стоит использовать права суперпользователя
2. Подготовьте [backend](https://www.terraform.io/docs/language/settings/backends/index.html) для Terraform:  
   а. Рекомендуемый вариант: S3 bucket в созданном ЯО аккаунте(создание бакета через TF)
   б. Альтернативный вариант:  [Terraform Cloud](https://app.terraform.io/)
3. Создайте конфигурацию Terrafrom, используя созданный бакет ранее как бекенд для хранения стейт файла. Конфигурации Terraform для создания сервисного аккаунта и бакета и основной инфраструктуры следует сохранить в разных папках.
4. Создайте VPC с подсетями в разных зонах доступности.
5. Убедитесь, что теперь вы можете выполнить команды `terraform destroy` и `terraform apply` без дополнительных ручных действий.
6. В случае использования [Terraform Cloud](https://app.terraform.io/) в качестве [backend](https://www.terraform.io/docs/language/settings/backends/index.html) убедитесь, что применение изменений успешно проходит, используя web-интерфейс Terraform cloud.

Ожидаемые результаты:

1. Terraform сконфигурирован и создание инфраструктуры посредством Terraform возможно без дополнительных ручных действий, стейт основной конфигурации сохраняется в бакете или Terraform Cloud
2. Полученная конфигурация инфраструктуры является предварительной, поэтому в ходе дальнейшего выполнения задания возможны изменения.

---
### Создание Kubernetes кластера

На этом этапе необходимо создать [Kubernetes](https://kubernetes.io/ru/docs/concepts/overview/what-is-kubernetes/) кластер на базе предварительно созданной инфраструктуры.   Требуется обеспечить доступ к ресурсам из Интернета.

Это можно сделать двумя способами:

1. Рекомендуемый вариант: самостоятельная установка Kubernetes кластера.  
   а. При помощи Terraform подготовить как минимум 3 виртуальных машины Compute Cloud для создания Kubernetes-кластера. Тип виртуальной машины следует выбрать самостоятельно с учётом требовании к производительности и стоимости. Если в дальнейшем поймете, что необходимо сменить тип инстанса, используйте Terraform для внесения изменений.  
   б. Подготовить [ansible](https://www.ansible.com/) конфигурации, можно воспользоваться, например [Kubespray](https://kubernetes.io/docs/setup/production-environment/tools/kubespray/)  
   в. Задеплоить Kubernetes на подготовленные ранее инстансы, в случае нехватки каких-либо ресурсов вы всегда можете создать их при помощи Terraform.
2. Альтернативный вариант: воспользуйтесь сервисом [Yandex Managed Service for Kubernetes](https://cloud.yandex.ru/services/managed-kubernetes)  
  а. С помощью terraform resource для [kubernetes](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs/resources/kubernetes_cluster) создать **региональный** мастер kubernetes с размещением нод в разных 3 подсетях      
  б. С помощью terraform resource для [kubernetes node group](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs/resources/kubernetes_node_group)
  
Ожидаемый результат:

1. Работоспособный Kubernetes кластер.
2. В файле `~/.kube/config` находятся данные для доступа к кластеру.
3. Команда `kubectl get pods --all-namespaces` отрабатывает без ошибок.

---
### Создание тестового приложения

Для перехода к следующему этапу необходимо подготовить тестовое приложение, эмулирующее основное приложение разрабатываемое вашей компанией.

Способ подготовки:

1. Рекомендуемый вариант:  
   а. Создайте отдельный git репозиторий с простым nginx конфигом, который будет отдавать статические данные.  
   б. Подготовьте Dockerfile для создания образа приложения.  
2. Альтернативный вариант:  
   а. Используйте любой другой код, главное, чтобы был самостоятельно создан Dockerfile.

Ожидаемый результат:

1. Git репозиторий с тестовым приложением и Dockerfile.
2. Регистри с собранным docker image. В качестве регистри может быть DockerHub или [Yandex Container Registry](https://cloud.yandex.ru/services/container-registry), созданный также с помощью terraform.

---
### Подготовка cистемы мониторинга и деплой приложения

Уже должны быть готовы конфигурации для автоматического создания облачной инфраструктуры и поднятия Kubernetes кластера.  
Теперь необходимо подготовить конфигурационные файлы для настройки нашего Kubernetes кластера.

Цель:
1. Задеплоить в кластер [prometheus](https://prometheus.io/), [grafana](https://grafana.com/), [alertmanager](https://github.com/prometheus/alertmanager), [экспортер](https://github.com/prometheus/node_exporter) основных метрик Kubernetes.
2. Задеплоить тестовое приложение, например, [nginx](https://www.nginx.com/) сервер отдающий статическую страницу.

Способ выполнения:
1. Воспользоваться пакетом [kube-prometheus](https://github.com/prometheus-operator/kube-prometheus), который уже включает в себя [Kubernetes оператор](https://operatorhub.io/) для [grafana](https://grafana.com/), [prometheus](https://prometheus.io/), [alertmanager](https://github.com/prometheus/alertmanager) и [node_exporter](https://github.com/prometheus/node_exporter). Альтернативный вариант - использовать набор helm чартов от [bitnami](https://github.com/bitnami/charts/tree/main/bitnami).

2. Если на первом этапе вы не воспользовались [Terraform Cloud](https://app.terraform.io/), то задеплойте и настройте в кластере [atlantis](https://www.runatlantis.io/) для отслеживания изменений инфраструктуры. Альтернативный вариант 3 задания: вместо Terraform Cloud или atlantis настройте на автоматический запуск и применение конфигурации terraform из вашего git-репозитория в выбранной вами CI-CD системе при любом комите в main ветку. Предоставьте скриншоты работы пайплайна из CI/CD системы.

Ожидаемый результат:
1. Git репозиторий с конфигурационными файлами для настройки Kubernetes.
2. Http доступ на 80 порту к web интерфейсу grafana.
3. Дашборды в grafana отображающие состояние Kubernetes кластера.
4. Http доступ на 80 порту к тестовому приложению.
---
### Установка и настройка CI/CD

Осталось настроить ci/cd систему для автоматической сборки docker image и деплоя приложения при изменении кода.

Цель:

1. Автоматическая сборка docker образа при коммите в репозиторий с тестовым приложением.
2. Автоматический деплой нового docker образа.

Можно использовать [teamcity](https://www.jetbrains.com/ru-ru/teamcity/), [jenkins](https://www.jenkins.io/), [GitLab CI](https://about.gitlab.com/stages-devops-lifecycle/continuous-integration/) или GitHub Actions.

Ожидаемый результат:

1. Интерфейс ci/cd сервиса доступен по http.
2. При любом коммите в репозиторие с тестовым приложением происходит сборка и отправка в регистр Docker образа.
3. При создании тега (например, v1.0.0) происходит сборка и отправка с соответствующим label в регистри, а также деплой соответствующего Docker образа в кластер Kubernetes.

---
## Что необходимо для сдачи задания?

1. Репозиторий с конфигурационными файлами Terraform и готовность продемонстрировать создание всех ресурсов с нуля.
2. Пример pull request с комментариями созданными atlantis'ом или снимки экрана из Terraform Cloud или вашего CI-CD-terraform pipeline.
3. Репозиторий с конфигурацией ansible, если был выбран способ создания Kubernetes кластера при помощи ansible.
4. Репозиторий с Dockerfile тестового приложения и ссылка на собранный docker image.
5. Репозиторий с конфигурацией Kubernetes кластера.
6. Ссылка на тестовое приложение и веб интерфейс Grafana с данными доступа.
7. Все репозитории рекомендуется хранить на одном ресурсе (github, gitlab)


## ПРОЦЕСС ВЫПОЛНЕНИЯ ДИПЛОМНОГО ПРАКТИКУМА

* [Этапы:](#Этапы)
     * [Создаем облачную инфраструктуру](#Создаем-облачную-инфраструктуру)
     * [Создаем Kubernetes кластер](#Создаем-Kubernetes-кластер)
     * [Создаем тестовое приложение](#Создаем-тестовое-приложение)
     * [Подготавливаем cистему мониторинга и деплой приложения](#Подготавливаем-cистему-мониторинга-и-деплой-приложения)
     * [Устанавливаем и настраиваем CICD](#Устанавливаем-и-настраиваем-CICD)

### Этапы:

### Создаем облачную инфраструктуру

1. Создаем аккаунт SA с ролью editor. Для автоматизации процесса был написан bash скрипт:

```
#!/bin/bash

#Создание сервис-аккаунта
yc iam service-account create --name sa

#Присваиваем переменные
sa_id="$(yc iam service-account list | grep 'sa' | awk '{print $2}')"
folder_id="$(yc config list | grep 'folder-id' | awk '{print $2}')"
cloud_id="$(yc config list | grep 'cloud-id' | awk '{print $2}')"

#Добавление роли сервис аккаунту
yc resource-manager folder add-access-binding $folder_id --role editor --subject serviceAccount:$sa_id

#Создание iam-ключей статический ключ доступа и ключа для доступа к облаку с записью в файлы
yc iam access-key create --service-account-id $sa_id --format json > ./access-key.json
yc iam key create --service-account-id $sa_id --folder-id $folder_id --output key.json

#Создание и конфигурирование профиля
yc config profile create sa
yc config set service-account-key key.json
yc config set cloud-id $cloud_id
yc config set folder-id $folder_id

#Экспорт необходимых переменнных окружения
export YC_CLOUD_ID="$(yc config get cloud-id)"
export YC_FOLDER_ID="$(yc config get folder-id)"
export AWS_ACCESS_KEY="$(grep 'key_id' ./access-key.json | awk '{print $2}' | tr -d \")"
export AWS_SECRET_KEY="$(grep 'secret' ./access-key.json | awk '{print $2}' | tr -d \")"
export TF_VAR_account_id=$sa_id
export TF_VAR_folder_id="$(yc config get cloud-id)"
export TF_VAR_cloud_id="$(yc config get folder-id)"
export TF_VAR_token="**********************************"
```

2. Создание бакета для хранения в нем состояния инфраструктуры.

Создаем tf файлы:

[providers.tf](https://github.com/Tourker/mydiplom-IAC/blob/main/prepare/providers.tf)
[bucket.tf](https://github.com/Tourker/mydiplom-IAC/blob/main/prepare/bucket.tf)
[variables.tf](https://github.com/Tourker/mydiplom-IAC/blob/main/prepare/variables.tf)

Скриншот созданного бакета:

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/cloud/1.jpg)

Инициализация бэкенда из бакета

Добавил в provider.tf блок backend для инициалазции в бакете

```
backend "s3" {
    endpoint = "https://storage.yandexcloud.net"
    bucket = "tfstate-bucket-diplom"
    region = "ru-central1"
    key    = "state/terraform.tfstate"

    skip_region_validation      = true
    skip_credentials_validation = true
    #skip_requesting_account_id  = true # Необходимая опция Terraform для версии 1.6.1 и старше.
    #skip_s3_checksum            = true # Необходимая опция при описании бэкенда для Terraform версии 1.6.3 и старше.

  }
```

[providers.tf](https://github.com/Tourker/mydiplom-IAC/blob/main/providers.tf)

Скриншот инициализации backend:

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/cloud/2.jpg)

Скриншот загруженного tfstate:

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/cloud/3.jpg)

3. Создаем VPC в разных зонах доступности

[network.tf](https://github.com/Tourker/mydiplom-IAC/blob/main/network.tf)

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/cloud/4.jpg)

4. Убеждаемся, что apply работает без каких-либо действий:

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/cloud/5.jpg)

### Создаем Kubernetes кластер

1. Для создания кластера у нас будет создано 4 виртуальные машины: 1 master и 3 workers ноды. Для масштабируемости был выбран вариант создания через группы виртуальных машин с привязкой целевых групп с балансировщиками нагрузки.

Создаем tf конфигурации с необходимыми ресурсами:

[nodes.tf](https://github.com/Tourker/mydiplom-IAC/blob/main/nodes.tf), [lb.tf](https://github.com/Tourker/mydiplom-IAC/blob/main/lb.tf), [providers.tf](https://github.com/Tourker/mydiplom-IAC/blob/main/providers.tf), [cloud-init.yaml](https://github.com/Tourker/mydiplom-IAC/blob/main/cloud-init.yaml)

Созданные ресурсы в YC:

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/cloud/6.jpg)

2. Для установки и настройки кластера будут использоваться свои ansible-конфигурации и так же ansible-playbook - Kubespray:

Предварительно необходимо подготовить шаблоны для автоматизации процесса установки:

Для начала создадим ansible.tf, в котором описан ресурс для формирования шаблона, а также сам шаблон inventory.tftpl, который будет собирать данные с IP-адресами и формировать необходимый конфиг файл.

[ansible.tf](https://github.com/Tourker/mydiplom-IAC/blob/main/ansible.tf)

[iventory.tftpl](https://github.com/Tourker/mydiplom-IAC/blob/main/inventory.tftpl)

После применения у нас добавляется файл inventory.yml в директорию ansible:

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/cluster/1.jpg)

Также создадим шаблон hosts.tftpl для формирования inventory master ноды, который будет размещаться в директории ansible/inventory:

[hosts.tftpl](https://github.com/Tourker/mydiplom-IAC/blob/main/hosts.tftpl)

Полученный файл hosts.yml:

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/cluster/2.jpg)

Переходим к установке предварительных пакетов с помощью ansible. Для этого были написаны ansible-конфигурации и размещены в каталоге ansible:

[prepare.yaml](https://github.com/Tourker/mydiplom-IAC/blob/main/ansible/prepare.yaml), [prepare-kubespray.yaml](https://github.com/Tourker/mydiplom-IAC/blob/main/ansible/prepare-kubespray.yaml), [kube-conf.yaml](https://github.com/Tourker/mydiplom-IAC/blob/main/ansible/kube-conf.yaml)

Для автоматизации процесса был написал скрипт установки playbook'ов:

start_install_kube.sh:

```
#!/bin/bash

host_ansible="$(grep 'ansible_host:' ./ansible/inventory/hosts.yml | awk '{print $2}' | tr -d \")"
cd ansible/
ansible-playbook -i inventory/hosts.yml prepare.yaml
ansible-playbook -i inventory/hosts.yml prepare-kubespray.yaml
ssh ubuntu@"$host_ansible" "cd kubespray/; /home/ubuntu/.local/bin/ansible-playbook -i inventory/mycluster/inventory.yml cluster.yml -b &"
ansible-playbook -i inventory/hosts.yml kube-conf.yaml

```
![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/cluster/3.jpg)

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/cluster/4.jpg)

Подключаемся к кластеру и убеждаемся что кластер развернут:

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/cluster/5.jpg)

### Создаем тестовое приложение

1. Создаем Dockerfile для сборки нашего приложения - одностраничного сайта в nginx 

Dockerfile:

```
FROM nginx:latest
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```
2. Собираем образ, логинимся в dockerhub и пушим образ в registry:

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/App/1.jpg)

3. Проверяем в registry dockerhub

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/App/2.jpg)

### Подготавливаем cистему мониторинга и деплой приложения

1. Систему мониторинга prometheus + grafana будем устанавливать с помощью helm chart - kube-promtheus-stack

Для этого вначале добавляем helm-репозиторий и устанавливаем helm chart в namespace - monitoring. Далее создаем NetworkPolicy и Service с типо NodePort для внешнего проброса через loadbalancer, созданный ранее при конфигурировании инфраструктуры.

Сылка на grafana.yaml

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/monitoring/1.jpg)

Заходим через IP-адресс loadbalancer указывая порт 3000

Login: admin
Password: prom-operator

Далее переходим в Dashboard/General/Kubernetes/Compute Resources/Cluster

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/monitoring/2.jpg)

Как видим все корректно отображается.

2. Деплоим наше приложение в кластер:

Создаем в default namespace файл app.yaml в котором располагается сам Deployment, NetworkPolicy и Service с типом NodePort:
```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-app
  labels:
    app: app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
      - name: diplom-app
        image: tourk/diplom-app:latest
        ports:
          - containerPort: 80
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: diplom-policy
spec:
  podSelector:
    matchLabels:
      app: app
  ingress:
  - from:
    - ipBlock:
        cidr: 0.0.0.0/0
    ports:
    - port: 80
      protocol: TCP
  policyTypes:
  - Ingress
---
apiVersion: v1
kind: Service
metadata:
  name: diplom-svc
spec:
  selector:
    app: app
  ports:
  - name: app-http
    protocol: TCP
    port: 80
    nodePort: 31080
    targetPort: 80
  type: NodePort

```
Деплоим приложение в кластер:

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/monitoring/3.jpg)

Заходим по IP-адресу на стандартном порту 80:

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/monitoring/4.jpg)

Смотрим в системе мониторинга, что Deployment появился:

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/monitoring/5.jpg)


### Устанавливаем и настраиваем CICD

Для построения CI/CD pipeline воспользуемся сервисом Managed Service for GitLab

1. В Yandex Cloud поднял инстанс:

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/CICD/app/1.jpg)

2. Создал проект mydiplom:

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/CICD/app/2.jpg)

3. Загрузил файлы проекта:

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/CICD/app/3.jpg)

4. Для работы в gitlab ci нам необходимо создать и законнектить gitlab-runner, выдать права rbac и задеплоить его в кластер. Для этих целей будем использовать helm chart:

1) Добавляем репозиторий и обновляем:

```
helm repo add gitlab https://charts.gitlab.io
helm repo update gitlab
```

2) Создаем файл конфигурации gitlab-value.yaml

[values-gitlab.yaml](https://github.com/Tourker/mydiplom-app/blob/main/gitlab_ci/values-gitlab.yaml)

3) Устанавливаем helm chart с подготовленным ранее value в namespace gitlab:

```
helm install --namespace gitlab gitlab-runner -f values-gitlab.yaml gitlab/gitlab-runner --create-namespace --set rbac.create=true
```

4) Подготавливаем yaml файл для предоставляения прав для runner'а и деплоим его в кластер:

[gitlab-runner.yaml](https://github.com/Tourker/mydiplom-app/blob/main/gitlab_ci/gitlab-runner.yaml)

5) Проверяем коннект раннера в gitlab:

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/CICD/app/4.jpg)

Также для выполнения команд kubectl в pipeline нам необходим gitlab agent:

1) Добавляем в директории проекта файл .gitlab/agents/agent/config.yaml с контекстом:

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/CICD/app/5.jpg)

2) Деплоим с помощью helm агент в кластер:

```
helm upgrade --install agent gitlab/gitlab-agent \
    --namespace gitlab-agent-agent \
    --create-namespace \
    --set image.tag=v17.4.2 \
    --set config.token=****************************** \
    --set config.kasAddress=wss://tourk.gitlab.yandexcloud.net/-/kubernetes-agent/
```

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/CICD/app/6.jpg)

3) Проверяем коннет агента в gitlab:

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/CICD/app/7.jpg)

5. Можем приступать к созданию pipeline:

В качестве registry контейнеров использовал dockerhub. Также для этапа BUILD использовался kaniko для сборки docker-образа и пуша его в registry.

1) Перед написанием pipeline необходимо задать необходимые переменные со скрытыми данными:

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/CICD/app/8.jpg)

2) Пишем Pipeline - при любом коммите в main ветку создается образ с тегом ветки и деплоится в кластер с перезагрузкой пода.

.gitlab-ci.yml

```
stages:
  - build
  - deploy
build:
  image:
    name: gcr.io/kaniko-project/executor:v1.9.0-debug
    entrypoint: [""]
  before_script:
    - AUTH=$(echo -n ${DOCKER_HUB_USER}:${DOCKER_HUB_PASSWORD} | base64)
    - cp -f "${CI_PROJECT_DIR}/config.json" /kaniko/.docker/config.json
    - sed -i "s/PLACE_HOLDER/$AUTH/" /kaniko/.docker/config.json
  stage: build
  script:
    - |
      /kaniko/executor --context "${CI_PROJECT_DIR}" --dockerfile "${CI_PROJECT_DIR}/Dockerfile" --destination "${DOCKER_HUB_USER}/${DOCKER_IMAGE_NAME}:${CI_COMMIT_REF_NAME}"
deploy:
  image:
    name: bitnami/kubectl:latest
  stage: deploy
  script:
    - sed -i "s/__VERSION__/${CI_COMMIT_REF_NAME}/" app.yaml
    - kubectl config get-contexts
    - kubectl config use-context tourk/mydiplom:agent
    - kubectl get pods -n default
    - kubectl apply -f app.yaml -n default
    - kubectl rollout restart deployment deployment-app -n default

```

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/CICD/app/9.jpg)

3) Проверяем работу:

- Первый запуск

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/CICD/app/10.jpg)

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/CICD/app/11.jpg)

- Меняем в index.html версию:

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/CICD/app/12.jpg)

Как видно все успешно деплоится и обновляется

6. Для мониторинга и управления инфраструктуры создадим pipeline в Gitlab CI:

1) Создаем закрытый проект terraform и добавляем туда файлы с конфигурациями:

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/CICD/terraform/1.jpg)

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/CICD/terraform/2.jpg)

2) Добавляем ранее задеплоиный gitlab-runner:

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/CICD/terraform/3.jpg)

3) Добавляем переменные:

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/CICD/terraform/4.jpg)

4) Создаем pipeline и проверяем работу:

.gitlab-ci.yml

```
image:
    name: registry.gitlab.com/gitlab-org/gitlab-build-images:terraform
    entrypoint:
        - '/usr/bin/env'
        - 'PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin'
cache:
    paths:
        - .terraform
variables:
    YC_SERVICE_ACCOUNT_KEY_FILE: /tmp/key.json

before_script:
  - |
    cat <<EOF >> ~/.terraformrc
    provider_installation {
        network_mirror {
            url = "https://terraform-mirror.yandexcloud.net/"
            include = ["registry.terraform.io/*/*"]
        }
        direct {
            exclude = ["registry.terraform.io/*/*"]
        }
    }
    EOF
  - terraform init -backend-config="access_key=${AWS_ACCESS_KEY_ID}" -backend-config="secret_key=${AWS_SECRET_ACCESS_KEY}"
  - echo $YC_KEY > /tmp/key.json

stages:
    - validate
    - plan
    - apply
validate:
    stage: validate
    script:
        - terraform validate
plan:
    stage: plan
    script:
        - terraform plan -out="planfile"
    artifacts:
        paths:
            - planfile
apply:
    stage: apply
    script:
        - terraform apply -auto-approve "planfile"
    when: manual

```

Запущенные Job'ы:

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/CICD/terraform/5.jpg)

Сформированные артефакты:

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/CICD/terraform/6.jpg)

Отображение успешно выполенной Job'ы:

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/CICD/terraform/7.jpg)

Проверим запуск pipeline на изменении в инфраструктуре. Для теста добавим в файл networks.tf еще одну подсеть и посмотрим как отрабатывается pipeline:

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/CICD/terraform/8.jpg)

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/CICD/terraform/9.jpg)

![Cкриншот](https://github.com/Tourker/mydiplom-desc/blob/main/img/CICD/terraform/10.jpg)

### Репозитории 

[Репозиторий с описание работы](https://github.com/Tourker/mydiplom-desc)

[Репозиторий с приложением, манифестами и пайплайнами](https://github.com/Tourker/mydiplom-app)

[Репозиторий с инфраструктурой](https://github.com/Tourker/mydiplom-IAC)

[Ссылка на ansible-конфигурации](https://github.com/Tourker/mydiplom-IAC/tree/main/ansible)

[Ссылка на личный репозиторий GITLAB в YC](https://tourk.gitlab.yandexcloud.net/)

[Ссылка на развернутое приложение](http://130.193.56.59)

[Ссылка на GRAFANA](http://130.193.56.59:3000) - admin/prom-operator

[Ссылка на Registry dockerhub](https://hub.docker.com/repository/docker/tourk/diplom-app/general)