# Дипломный практикум в Yandex.Cloud
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


### Вся конфигурация 
[Конфигурация Terraform, kubspray, скрипты](https://github.com/rbudarin/src-for-diplom)

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
2. Подготовьте [backend](https://developer.hashicorp.com/terraform/language/backend) для Terraform:  
   а. Рекомендуемый вариант: S3 bucket в созданном ЯО аккаунте(создание бакета через TF)
   б. Альтернативный вариант:  [Terraform Cloud](https://app.terraform.io/)

## Смотрим cloud-id:... folder-id: ...
yc config list
## Настройкая зеркала
```
##______________Config__________________
provider_installation {
  network_mirror {
    url = "https://terraform-mirror.yandexcloud.net/"
    include = ["registry.terraform.io/*/*"]
  }
  direct {
    exclude = ["registry.terraform.io/*/*"]
  }
}
##_______________________________________
```
## Создаем проект:
```
mkdir -p terraform/storage
touch terraform/storage/{main.tf,variables.tf,output.tf}
touch terraform/{backend.tf,vpc.tf,variables.tf,providers.tf}

Смотрим структуру:
tree ./terraform
```
![tree](https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/tree.png)

## Создааем сервисные аккаунты
```
resource "yandex_iam_service_account" "terraform" {
  name        = "tf-srv-account"
  description = "Service account for Terraform"
}

resource "yandex_resourcemanager_folder_iam_binding" "editor" {
  folder_id = var.yc_folder_id
  role      = "editor"
  members   = [
    "serviceAccount:${yandex_iam_service_account.terraform.id}",
  ]
}

resource "yandex_resourcemanager_folder_iam_binding" "storage_admin" {
  folder_id = var.yc_folder_id
  role      = "storage.admin"
  members   = [
    "serviceAccount:${yandex_iam_service_account.terraform.id}",
  ]
}
```

### Подготовливаем s3-bucket под backend
```
## Static access key for bucket
resource "yandex_iam_service_account_static_access_key" "sa-static-key" {
  service_account_id = yandex_iam_service_account.terraform.id
  description        = "Static access key for Terraform backend"
}

## Create bucket
resource "yandex_storage_bucket" "storage_bucket" {
  bucket     = "${var.bucket_name}-${formatdate("DD.MM.YYYY", timestamp())}" 
  access_key = yandex_iam_service_account_static_access_key.sa-static-key.access_key
  secret_key = yandex_iam_service_account_static_access_key.sa-static-key.secret_key
  
  default_storage_class = "STANDARD"
  acl           = "public-read"
  force_destroy = "true"
  max_size      = var.bucket_max_size 


  versioning {
    enabled = true
  }
}
```

## Собираем конфигурацию
```
terraform init
```
# Проверяем конфигурацию
```
terraform validate
```
# Создаем bucket под backend
```
terraform apply -auto-approve
```
# Посмотреть ключи
terraform output -json
---
terraform output access_key
terraform output secret_key

# Сохроняем информацию для backand 
```
echo -e "bucket = $(terraform output bucket_name)\naccess_key = $(terraform output access_key)\nsecret_key = $(terraform output secret_key)" > ../.backend.hcl
```
# Подключим файл с секретами backend при инициации инфраструктуры:
```
cd ../ && terraform init -backend-config=.backend.hcl
terraform init -upgrade
terraform apply -auto-approve
```
## Смотрим наш bucket
![bucket](https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/bucket.png)


3. Создайте конфигурацию Terrafrom, используя созданный бакет ранее как бекенд для хранения стейт файла. Конфигурации Terraform для создания сервисного аккаунта и бакета и основной инфраструктуры следует сохранить в разных папках.
4. Создайте VPC с подсетями в разных зонах доступности.

## Создаем VPC через foreach и locals
```
##______________VPC______________________
resource "yandex_vpc_network" "network" {
  name = "main-network"
}
```
# Создание подсетей с использованием locals
```
resource "yandex_vpc_subnet" "subnets" {
  for_each = local.subnets

  name           = "${each.key}"
  zone           = each.value.zone
  network_id     = yandex_vpc_network.network.id
  v4_cidr_blocks = each.value.v4_cidr_blocks
}
##_______________locals___________________
locals {
  subnets = {
    "subnet-a" = {
      zone           = "ru-central1-a"
      v4_cidr_blocks = ["192.168.1.0/24"]
    },
    "subnet-b" = {
      zone           = "ru-central1-b"
      v4_cidr_blocks = ["192.168.2.0/24"]
    },
    "subnet-d" = {
      zone           = "ru-central1-d"
      v4_cidr_blocks = ["192.168.3.0/24"]
    }
  }
}
##_______________________________________
```

![network-subnet](https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/network-subnet.png)

5. Убедитесь, что теперь вы можете выполнить команды `terraform destroy` и `terraform apply` без дополнительных ручных действий.

![destroy](https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/destroy.png)

6. В случае использования [Terraform Cloud](https://app.terraform.io/) в качестве [backend](https://developer.hashicorp.com/terraform/language/backend) убедитесь, что применение изменений успешно проходит, используя web-интерфейс Terraform cloud.

Ожидаемые результаты:

1. Terraform сконфигурирован и создание инфраструктуры посредством Terraform возможно без дополнительных ручных действий, стейт основной конфигурации сохраняется в бакете или Terraform Cloud

![bucket01](https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/bucket01.png)
![bucket02](https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/bucket02.png)

2. Полученная конфигурация инфраструктуры является предварительной, поэтому в ходе дальнейшего выполнения задания возможны изменения.

---
### Создание Kubernetes кластера

На этом этапе необходимо создать [Kubernetes](https://kubernetes.io/ru/docs/concepts/overview/what-is-kubernetes/) кластер на базе предварительно созданной инфраструктуры.   Требуется обеспечить доступ к ресурсам из Интернета.

Это можно сделать двумя способами:

1. Рекомендуемый вариант: самостоятельная установка Kubernetes кластера.  
   а. При помощи Terraform подготовить как минимум 3 виртуальных машины Compute Cloud для создания Kubernetes-кластера. Тип виртуальной машины следует выбрать самостоятельно с учётом требовании к производительности и стоимости. Если в дальнейшем поймете, что необходимо сменить тип инстанса, используйте Terraform для внесения изменений.  
![vm_cloud](https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/vm_cloud.png)
![vm_cloud01](https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/vm_cloud01.png)

   б. Подготовить [ansible](https://www.ansible.com/) конфигурации, можно воспользоваться, например [Kubespray](https://kubernetes.io/docs/setup/production-environment/tools/kubespray/)  
   в. Задеплоить Kubernetes на подготовленные ранее инстансы, в случае нехватки каких-либо ресурсов вы всегда можете создать их при помощи Terraform.
3. Альтернативный вариант: воспользуйтесь сервисом [Yandex Managed Service for Kubernetes](https://cloud.yandex.ru/services/managed-kubernetes)  
  а. С помощью terraform resource для [kubernetes](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs/resources/kubernetes_cluster) создать **региональный** мастер kubernetes с размещением нод в разных 3 подсетях      
  б. С помощью terraform resource для [kubernetes node group](https://registry.terraform.io/providers/yandex-cloud/yandex/latest/docs/resources/kubernetes_node_group)
  
Ожидаемый результат:

1. Работоспособный Kubernetes кластер.
2. В файле `~/.kube/config` находятся данные для доступа к кластеру.
3. Команда `kubectl get pods --all-namespaces` отрабатывает без ошибок.

# Клонируем репозиторий Kubespray
```
git clone https://github.com/kubernetes-sigs/kubespray.git
cd kubespray
```
# Создаем виртуальное пространство и Устанавливаем зависимости
```
python3 -m venv ~/venv3.11
source ~/venv3.11/bin/activate
sudo apt install -y python3 python3-pip
```
# Установка зависимостей
```
pip3 install -r requirements.txt
```
# Копируем пример инвентаря
```
cp -rfp inventory/sample inventory/mycluster
```
# Копируем наш шаблон
```
cp -rfp ansible/hosts.cfg kubespray/inventory/mycluster/inventory.ini
```
![k8s](https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/k8s.png)

# Запуск развертывания Kubernetes
ansible-playbook -i inventory/mycluster/inventory.ini -u ubuntu --become --become-user=root cluster.yml

![k8s1](https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/k8s1.png)
![kube_config](https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/kube_config.png)
![kuber_cluster](https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/kuber_cluster.png)

---
### Создание тестового приложения
mkdir -p test-app
touch test-app/{Dockerfile,index.html,nginx.conf}

# Запускаем образ и проверяем
docker run -d -p 8080:80 --name nginx rbudarin/nginx:v0.1 
curl -i 127.0.0.1:8080

![docker_web_app01](https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/docker_web_app01.png)
![docker_web_app02](https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/docker_web_app02.png)

Для перехода к следующему этапу необходимо подготовить тестовое приложение, эмулирующее основное приложение разрабатываемое вашей компанией.


Способ подготовки:

1. Рекомендуемый вариант:  
   а. Создайте отдельный git репозиторий с простым nginx конфигом, который будет отдавать статические данные.  

[test-app](https://github.com/rbudarin/devops-diplom-yandexcloud/tree/main/src/test-app)

   б. Подготовьте Dockerfile для создания образа приложения.  
3. Альтернативный вариант:  
   а. Используйте любой другой код, главное, чтобы был самостоятельно создан Dockerfile.

Ожидаемый результат:

1. Git репозиторий с тестовым приложением и Dockerfile.

2. Регистри с собранным docker image. В качестве регистри может быть DockerHub или [Yandex Container Registry](https://cloud.yandex.ru/services/container-registry), созданный также с помощью terraform.

# Создаем три файла Dockerfile, index.html, nginx.conf и собираем образ
docker build -t rbudarin/nginx:v0.1 .
# Авторизируемся в docker hub
docker login -u rbudarin
# Создаем ветку и заливаем туда свой конфиг
docker push rbudarin/nginx:v0.1

![activate_dockerhub](https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/activate_dockerhub.png)
![docker_build](https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/docker_build.png)
![dockerhub_push](https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/dockerhub_push.png)
![docker_hub](https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/docker_hub.png)

---
### Подготовка cистемы мониторинга и деплой приложения

Уже должны быть готовы конфигурации для автоматического создания облачной инфраструктуры и поднятия Kubernetes кластера.  
Теперь необходимо подготовить конфигурационные файлы для настройки нашего Kubernetes кластера.

Цель:
1. Задеплоить в кластер [prometheus](https://prometheus.io/), [grafana](https://grafana.com/), [alertmanager](https://github.com/prometheus/alertmanager), [экспортер](https://github.com/prometheus/node_exporter) основных метрик Kubernetes.
2. Задеплоить тестовое приложение, например, [nginx](https://www.nginx.com/) сервер отдающий статическую страницу.

Способ выполнения:
1. Воспользоваться пакетом [kube-prometheus](https://github.com/prometheus-operator/kube-prometheus), который уже включает в себя [Kubernetes оператор](https://operatorhub.io/) для [grafana](https://grafana.com/), [prometheus](https://prometheus.io/), [alertmanager](https://github.com/prometheus/alertmanager) и [node_exporter](https://github.com/prometheus/node_exporter). Альтернативный вариант - использовать набор helm чартов от [bitnami](https://github.com/bitnami/charts/tree/main/bitnami).

# Создайте манифесты для развертывания мониторинга:
```
mkdir -p monitoring && cd !$
touch values.yaml
```
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```
##_________________values___________________
grafana:
  enabled: true
  adminPassword: "P@$$w0rD!23"
  service:
    portName: http-web
    type: NodePort
    nodePort: 30081
##__________________________________________
```
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack --create-namespace -n monitoring -f values
kubectl get svc -n monitoring kube-prometheus-stack-grafana -o wide
kubectl describe svc -n monitoring kube-prometheus-stack-grafana
kubectl delete ns monitoring
```
![grafana01](https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/grafana01.png)
![gf_metriki01]https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/gf_metriki01.png)
![gf_metriki02]https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/gf_metriki02.png)

Создадим файл deployment.yaml для развёртывания приложения в Kubernetes:
```
mkdir test-app
cd !$
touch manifest
```

```
##_______________manifest___________________
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-app
  template:
    metadata:
      labels:
        app: nginx-app
    spec:
      containers:
      - name: nginx
        image: rbudarin/nginx:v0.1
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  ports:
  - protocol: TCP
    port: 80
  selector:
    app: nginx-app
##_______________________________________
```

# Создадим namespace и задиплоим
```
kubectl create ns test-app
kubectl apply -f manifest -n test-app
```
![test-app](https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/test-app.png)
![test_app01](https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/test_app01.png)
![test-app-deploy](https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/test-app-detest-app-deployploy.png)

kubectl delete ns test-app

## Подготовка Helm чарта для созданного приложения
## через helm
```
mkdir -p test-app/templates
touch test-app{chart.yaml,values.yaml}
touch test-app/templates/deployment.yaml
```

```
##_______________chart___________________
echo 'apiVersion: v2
name: nginx-app
description: A Helm chart for deploying Nginx test application
version: "0.1"
appVersion: "0.1"' >  test-app/Chart.yaml
##_______________values___________________
echo 'replicaCount: 2
app:
  name: nginx-app

image:
  repository: rbudarin/nginx
  tag: latest

service:
  type: NodePort
  port: 80
  nodePort: 30080' > test-app/values.yaml
##_______________deployment___________________

echo '---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.app.name }}
  labels:
    app: {{ .Values.app.name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Values.app.name }}
  template:
    metadata:
      labels:
        app: {{ .Values.app.name }}
    spec:
      containers:
      - name: webapp-container
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.app.name }}-service
spec:
  type: {{ .Values.service.type }}
  selector:
    app: {{ .Values.app.name }}
  ports:
    - protocol: TCP
      port: 80
      nodePort: {{ .Values.service.nodePort }}' > test-app/templates/deployment.yaml
##_______________________________________
```

## Проверяем
```
helm template test-app
```
## Выполним проверку конструктора helm чарта:
```
helm lint test-app
```
## Выполним запуск на исполнение helm чарта:
```
helm upgrade --install test-app test-app --set image.tag=v0.1
```
## Проверим развернутые ресурсы:
```
helm ls
```
![helm_chart1](https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/helm_chart1.png)
![helm_chart2](https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/helm_chart2.png)
![helm_chart3](https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/helm_chart3.png)
![helm_chart4](https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/helm_chart4.png)

# Проверяем в мониторинге появилось наше приложения
![grafan_monitoring_test-app](https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/grafan_monitoring_test-app.png)

## Удалим развернутые helm ресурсы с кластера Kubernetes:
```
helm uninstall test-app
```
## Проверим что все удалилось 
```
helm ls
```

### Деплой инфраструктуры в terraform pipeline

1. Если на первом этапе вы не воспользовались [Terraform Cloud](https://app.terraform.io/), то задеплойте и настройте в кластере [atlantis](https://www.runatlantis.io/) для отслеживания изменений инфраструктуры. Альтернативный вариант 3 задания: вместо Terraform Cloud или atlantis настройте на автоматический запуск и применение конфигурации terraform из вашего git-репозитория в выбранной вами CI-CD системе при любом комите в main ветку. Предоставьте скриншоты работы пайплайна из CI/CD системы.

Ожидаемый результат:
1. Git репозиторий с конфигурационными файлами для настройки Kubernetes.
2. Http доступ на 80 порту к web интерфейсу grafana.
3. Дашборды в grafana отображающие состояние Kubernetes кластера.
4. Http доступ на 80 порту к тестовому приложению.
5. Atlantis или terraform cloud или ci/cd-terraform
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

# Создаем репозиторий 
https://github.com/rbudarin/test-app-deploy
![creat_repository](https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/creat_repository.png)

## Добавим в репозиторий атрибуты доступа в Docker Hub
Settings / Secrets and variables / Actions / Secrets / New repository secret
DockerHub_User
DockerHub_Password
![repository_secret](https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/repository_secret.png)

## Добавим в GitHub-профиль публичный ключ ssh и проверим возможность работы с ним через консоль
ssh-keygen -t ed25519
/home/user/.ssh/git

cat ~/.ssh/git.pub
## И добавим этот ключ в git
GitHub / Settings / SSH and GPG keys / New SSH key

## Проверяем
ssh -T git@github.com -i ~/.ssh/git
ssh-add ~/.ssh/git
ssh -T git@github.com

# Создадим директорию для работы с GitHub Actions
```
mkdir -p actions/.github/workflows
cd !$
```
Создадим файл для описания процесса развёртывания в GitHub Actions:
```
##_____________workflows________________________
name: Build and Deploy

on:
  push:
    branches: [ main ]
    tags: [ 'v*' ]
    
jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Login to DockerHub
      uses: docker/login-action@v1
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_PASSWORD }}

    - name: Build and push Docker image
      uses: docker/build-push-action@v2
      with:
        context: .
        push: true
        tags: rbudarin/nginx:${{ github.ref_name }}
        
    - name: Setup kubeconfig
      run: echo '${{ secrets.KUBECONFIG_FILE }}' > kubeconfig
      
    - name: Install Kubernetes CLI
      run: |
        curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
        chmod +x kubectl
        sudo mv kubectl /usr/local/bin/

    - name: Delete old deployment
      run: |
        export KUBECONFIG=kubeconfig
        kubectl delete deploy nginx-app || true

    - name: Deploy to Kubernetes
      run: |
        export KUBECONFIG=kubeconfig
        kubectl apply -f deployment.yaml && kubectl apply -f service.yaml
  ##_______________________________________________
```

 ## структруа
 ![tree](https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/tree.png)

# Что бы работало с нашим k8s надо добавить KUBECONFIG_FILE (это ~/.kube/config)
 ![secret_k8s](https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/secret_k8s.png)
 
# Подключим созданный GitHub-репозиторий:
```
git init
git add .
git commit -m "commit v0.1"
git branch -M main
git remote add origin https://github.com/rbudarin/test-app-deploy.git
git push --force origin main
```
![docker_image](https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/docker_image.png)
![git01](https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/git01.png)
![git02](https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/git02.png)
![git03](https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/git03.png)
![git04](https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/git04.png)

# Проверяем в докере, все пушиться с тегом
![git05](https://github.com/rbudarin/devops-diplom-yandexcloud/blob/main/screen/git05.png)
[test-app-deploy](https://github.com/rbudarin/test-app-deploy)

# Docker Hub
## [docker_hub](https://hub.docker.com/repository/docker/rbudarin/nginx/tags)

---
## Что необходимо для сдачи задания?

1. Репозиторий с конфигурационными файлами Terraform и готовность продемонстрировать создание всех ресурсов с нуля.
2. Пример pull request с комментариями созданными atlantis'ом или снимки экрана из Terraform Cloud или вашего CI-CD-terraform pipeline.
3. Репозиторий с конфигурацией ansible, если был выбран способ создания Kubernetes кластера при помощи ansible.
4. Репозиторий с Dockerfile тестового приложения и ссылка на собранный docker image.
5. Репозиторий с конфигурацией Kubernetes кластера.
6. Ссылка на тестовое приложение и веб интерфейс Grafana с данными доступа.
7. Все репозитории рекомендуется хранить на одном ресурсе (github, gitlab)

