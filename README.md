# Ingvar78_infra
Ingvar78 Infra repository

<h1> 5. ДЗ Знакомство с облачной инфраструктурой. Yandex.Cloud </h1>

```
bastion_IP = 84.201.157.40
someinternalhost_IP = 10.130.0.21
testapp_IP = 84.201.130.72
testapp_port = 9292
```

<h2> 5.1 Самостоятельное задание </h2>
способ подключение к `someinternalhost` одной командой

опции ssh: -A пробрасывает авторизацию на удалённый сервер. -J - Jumphost, -l - login_name, -i - identity_file

[проброс по SSH](https://itsecforu.ru/2018/11/29/%D0%BA%D0%B0%D0%BA-%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B8%D1%82%D1%8C-%D0%B4%D0%BE%D1%81%D1%82%D1%83%D0%BF-%D0%BA-%D1%83%D0%B4%D0%B0%D0%BB%D0%B5%D0%BD%D0%BD%D0%BE%D0%BC%D1%83-%D1%81%D0%B5%D1%80%D0%B2/)

```
ssh -J appuser@<bastion_IP> appuser@<someinternalhost_IP>
**OR**
ssh -i ~/.ssh/appuser -A -J appuser@<bastion_IP> appuser@<someinternalhost_IP>
**OR**
ssh -l appuser -A -J appuser@<bastion_IP> <someinternalhost_IP>
```
подключение по hostname путем настройки .ssh/config ProxyCommand
```
Host someinternalhost
     HostName 10.130.0.21
     User appuser
     ProxyCommand ssh -W %h:%p appuser@84.201.157.40
```

<h2> 5.2 Создаем VPN-сервер для серверов Yandex.Cloud </h2>

На хосте **bastion** установлен и настроен VPN-сервер Pritunl.
```
Для подключения по vpn через GUI необходимо скачать файл конфигурации пользователя - *cloud-bastion.ovpn* и выполнить импорт конфигурации (запросит логин\пароль).

Pritunl имеет встроенную интеграцию с Let's encrypt.
Для использования валидного сертификата необходимо в настройках Pritunl "Lets Encrypt Domain" указать соответсвующий URL.
Сертификат сгенерируется автоматически на стороне сервиса sslip.io и привяжется к FQDN.
```
Lets Encrypt Domain ``https://84-201-157-40.sslip.io/``

<h1> 6. Основные сервисы Yandex Cloud </h1>

```
yc compute instance create \
  --name reddit-app1 \
  --hostname reddit-app1 \
  --memory=4 \
  --create-boot-disk image-folder-id=standard-images,image-family=ubuntu-1604-lts,size=10GB \
  --network-interface subnet-name=default-ru-central1-a,nat-ip-version=ipv4 \
  --metadata-from-file user-data=./reddit.yaml
```

<h1> 7. Модели управления инфраструктурой. Подготовка образов с помощью Packer  </h1>
<h2> 7.1 Сборка образов VM при помощи Packer</h2>

Создан шаблон "ubuntu16.json" для создания образа

Добавлены провиженеры rubby и mongodb

Создан образа. В процессе сборки образа  добавлена ("use_ipv4_nat": true), и таймауты запуска скриптов ("pause_before": "60s")

Создан инстанс на основе образа. Выполнен деплой reddit приложения с использованием скрипта из предыдущего задания

<h2> 7.2 Самостоятельные задания </h2> 

Выполнена параметризация шаблона (variables.json)

```
  packer build -var-file=variables.json ./ubuntu16.json
```

<h2> 7.3 Дополнительные задания </h2>

В шаблон добавлены дополнителный "provisioners" для деплоя приложения reddit, cоздан образ на основе базового

Сбокра образа:

```
  packer build -var-file=variables.json ./immutable.json
```

Создан скрипт create-reddit-vm.sh

Проверка работы http://84.201.131.37:9292/


<h1> 8. Практика IaC с использованием Terraform  </h1>

<h2> 8.1 Декларативное описание в виде кода инфраструктуры YC, требуемой для запуска тестового приложения, при помощи Terraform.</h2>

Изучена документация terraform

Определаны input переменные

Отформатированы все файлы через terraform fmt

Создан файл terraform.tfvars.example с примерами переменных

Создан файл lb.tf для **Load Balancer**, через outputs.tf выведены ip адреса инстансов и load balancer

Проверена работоспособность LB с несколькими аппами. Проверена работоспособность LB при выключении сервиса на одном из аппов.

Дополнен main.tf для возможности создавать n инстансов с приложениями, через изменение значения app_instances_count (в текущей конфигурации максимум 3 инстанса минимум 1), для создания большего числа инстансов необходимо внести изменения в outputs.tf

Создание инстансов: 

```
$ terraform apply -auto-approve

....

Apply complete! Resources: 0 added, 1 changed, 0 destroyed.

Outputs:

external_ip_address_app-1 = 130.193.50.49
external_ip_address_app-2 = 130.193.37.100
external_ip_address_app-3 = 130.193.36.65
reddit-balancer-address = [
  "84.201.159.22",
]
```

<h1> 10. Создание Terraform модулей для управления компонентами инфраструктуры. </h1>

Дла новых шаблона db.json и app.json для создания образа ВМ с предустановленными mongodb и ruby

```
packer build -var-file=variables.json ./app.json
packer build -var-file=variables.json ./db.json
```

<h2> 10.1 Модули </h2>

Конфигурация терраформа разбита на модули:
    app - для развертывания приложения
    db - для базы данных
    vpc - для сети

<h2> 10.2 Переиспользование модулей </h2>
Созданы 2 окружения stage и prod, использующие модули db. app и vpc

Для загрузки модулей выполнить: terraform get

Для деплоя необходимо из соответствующей директории (stage или prod) выполнить:

```
$ terraform init
$ terraform plan
$ terraform apply -auto-approve
$ terraform destroy -auto-approve
```
<h2> 10.3 Удалённое хранение стэйта </h2>

Для генерации/просмотра ключей воспользоваться командой

```
yc iam access-key create --service-account-name <name_sa> --description "this key is for my bucket"
yc iam access-key list --service-account-name <name_sa>
```

Добавлен storage-bucket.tf, backend.tf

TODO: Выполнить задания со *
