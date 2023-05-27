### Создание образа через Packer
#### Конфигурация сети
Перед созданием образа необходимо создать сеть и подсеть.

Создание облачной сети в каталоге:
```bash
yc vpc network create \
  --name my-yc-network
```

Создадим подсеть в облачной сети my-yc-network:
```bash
yc vpc subnet create \
  --name my-yc-subnet-a \
  --zone ru-central1-a \
  --range 10.1.2.0/24 \
  --network-name my-yc-network
```
<br>

### Конфигурация образа
Создаем packer-image.json с содержимым ниже:
```
{
  "builders": [
    {
      "type":      "yandex",
      "token":     "XXX",
      "folder_id": "XXX",
      "zone":      "ru-central1-a",

      "image_name":        "my-ubuntu-2004-{{isotime | clean_resource_name}}",
      "image_family":      "my-ubuntu-2004",
      "image_description": "my custom ubuntu-20.04",

      "source_image_family": "ubuntu-2004-lts",
      "subnet_id":           "XXX",
      "use_ipv4_nat":        true,
      "disk_type":           "network-hdd",
      "ssh_username":        "ubuntu"
    }
  ],
  "provisioners": [
    {
      "type": "shell",
      "inline": [
        "echo 'updating APT'",
        "sudo apt-get update -y",
        "sudo apt-get install -y net-tools wget zip unzip curl telnet htop",
        "curl -fsSL https://get.docker.com -o get-docker.sh",
        "sh get-docker.sh",
        "sudo ln -s /usr/libexec/docker/cli-plugins/docker-compose /usr/bin/",
        "sudo systemctl start docker & sudo systemctl enable docker",
        "sudo usermod -aG docker $USER"
      ]
    }
  ]
}
```
Где:<br>
"token" можно получить по запросу: https://oauth.yandex.ru/authorize?response_type=token&client_id=1a6990aa636648e9b2ef855fa7bec2fb
"folder_id" берем из вывода "yc config list"<br>
"subnet_id" берем из вывода "yc vpc subnet list"<br>
<br>

Проверяем на валидность:
```bash
packer validate packer-image.json
```
<br>

Запускаем сборку:
```bash
packer build packer-image.json
```
<br>

В консоле https://console.cloud.yandex.ru/ будет виден образ.
<br>

### Создание ВМ из образа
Создание ВМ:
```bash
yc compute instance create \
  --name vm-ubuntu \
  --hostname node01 \
  --zone=ru-central1-a \
  --create-boot-disk size=50GB,image-id=fd8u97r916g4torfvmm2 \
  --cores=2 \
  --memory=4G \
  --core-fraction=100 \
  --network-interface subnet-id=e9blfcs478crgqoml1t6,ipv4-address=auto,nat-ip-version=ipv4 \
  --ssh-key ~/.ssh/yacloud.pub
```
<br>

Подключение к ВМ:
```bash
ssh -i ~/.ssh/yacloud yc-user@158.160.37.ХХХ
```
<br>

### Удаление ресурсов
#### Образ
Посмотреть список образов:
```bash
yc compute image list
```

Удалить образ:
```bash
yc compute image delete --name <name-image>
```
<br>

#### ВМ
Посмотреть список ВМ:
```bash
yc compute instance list
```

Остановить ВМ:
```bash
yc compute instance stop <name-instance>
```

Удалить ВМ:
```bash
yc compute instance delete <name-vm>
```
<br>

#### Подсеть
Посмотреть список подсетей:
```bash
yc vpc subnet list
```

Удалить подсеть:
```bash
yc vpc subnet delete --name <name-network>
```
<br>

#### Сеть
Посмотреть список сетей:
```bash
yc vpc network list
```

Удалить подсеть:
```bash
yc vpc network delete <name-network>
```


