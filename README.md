## Развёртывание
Для полноценного развёртывания необходимы 3 компонента:
* `master` service 
* `volume` service
* `filer` service (для конфигурации необходим _filer.toml_ файл)

Компоненты могуть находиться как на одной машине, так и на разных. Взаимодействие между компонентами осуществляется по **gRPC**
## master
Является 'головным элементом' инфраструктуры, отвечает за выбор места для хранения данных, вызывает аллокацию у `volume` сервисов, отвечает за репликацию и связи между `filer` и `volume` сервисами
## volume
Является непосредственным хранилищем данных, связывается с `master` сервисом для подчинения ему и приёма вызовов для работы с файлами.
## filer
Является gateway - аггрегатором для высокоуровневой работой с системой. Для функционирования, необходима инфраструктура из `master` + `volume` и хранилище для хранения информации о файлах. Конфигурация хранилища находится в _filer.toml_, подробнее о filer sotore: https://github.com/seaweedfs/seaweedfs/wiki/Filer-Stores. 
Для конфигурации хранения необходим _filer.toml_ файл, в котором будет указан способ хранения данныъ о файлах. Пример:
```
[leveldb2]
enabled = true
dir = "/data/filerldb2"
```
Полное описание _filer.toml_: https://github.com/seaweedfs/seaweedfs/wiki/Filer-Setup
API для работы с `filer`: https://github.com/seaweedfs/seaweedfs/wiki/Filer-Server-API
# Docker-compose
Данный образ наиболее удобен, поскольку позволяет запускать и удирать несколько контейнеров одновременно
Базовый docker образ - `chrislusf/seaweedfs`
При логальной развёртке базового кластера можно использовать следующий `docker-compose.yaml` файл
```
version: '3.9'

services:
  master:
    image: chrislusf/seaweedfs
    ports:
      - 9333:9333 # hhtp port
      - 19333:19333 # grpc port
      - 9324:9324 # metrics port
    volumes:
        - /path/to/master_data/local:/path/to/master_data/container
    command: "master -ip=master -port=9333 -port.grpc=19333 -ip.bind=0.0.0.0 -metricsPort=9324 -mdir=/path/to/master_data/container" # важно указывать ip.bind=0.0.0.0
  volume:
    image: chrislusf/seaweedfs # use a remote image
    ports:
      - 8080:8080 # hhtp port
      - 18080:18080 # grpc port
      - 9325:9325 # metrics port
    volumes:
        - /path/to/volume_data/local:/path/to/volume_data/container
    command: 'volume -mserver="master:9333" -ip="volume" -port=8080 -port.grpc=18080 -ip.bind=0.0.0.0 -port=8080  -metricsPort=9325 -dir=/path/to/volume_data/container -dataCenter=dc -rack=rc' # важно указывать ip.bind=0.0.0.0
    depends_on:
      - master
  filer:
    image: chrislusf/seaweedfs # use a remote image
    ports:
      - 8888:8888 # hhtp port
      - 18888:18888 # grpc port
      - 9326:9326 # metrics port
    volumes:
        - /path/to/filer_config/filer.toml:/etc/seaweedfs/filer.toml
        - /path/to/filer_data/local:/path/to/filer_data/container
    command: 'filer -master="master:9333" -port=8888 -port.grpc=18888 -ip.bind=0.0.0.0 -metricsPort=9326' # важно указывать ip.bind=0.0.0.0
    depends_on:
      - master
      - volume
```
Вызовете ```sudo docker-compose up``` или ```sudo docker-compose up -d``` для старта всех 3 сервисов, ```sudo docker-compose down``` для остановки.
## Docker
Для запуска всех компонентов в разных docker контейнерах необходимо сделать общую сеть
```
docker network create -d bridge weed_network
```
Далее создаём `master`
```
docker run --name weed_m -d -p 9334:9334 -p 19334:19334 \
	--network="weed_network" \
  	-v ./data/master:/data/master \
	chrislusf/seaweedfs master \
	-ip="weed_m" \
	-ip.bind="0.0.0.0" \
	-port=9334 \
	-port.grpc=19334 \
	-mdir="/data/master"
```
Создаём `volume`
```
docker run --name weed_v -d -p 8082:8082 -p 18082:18082 \
	--network="weed_network" \
  	-v ./data/volume:/data/volume \
	chrislusf/seaweedfs volume \
	-mserver="weed_m:9334" \
	-ip="weed_v" \
	-ip.bind="0.0.0.0" \
	-port=8082 \
	-port.grpc=18082 \
	-dataCenter="dc1" \
	-rack="rc1" \
	-dir="/data/volume"
```
Создаём `filer` с публичным адресом, чтобы мы могли обратиться к нему с другой машины. Так же нам понадоьится `filer.toml`
```
docker run --name weed_f -d -p 8889:8889 -p 18889:18889 \
	--network="weed_network" \
  	-v ./filer.toml:/etc/seaweedfs/filer.toml \
	-v ./data/filer:/db \
	chrislusf/seaweedfs filer \
	-master="weed_m_2:9334" \
	-ip="publick_addr" \
	-ip.bind="0.0.0.0" \
	-port=8889 \
	-port.grpc=18889
```
# Добавление нового volume сервера к master
## Docker-compose
```
version: '3.9'

services:
    volume:
        image: chrislusf/seaweedfs # use a remote image
    ports:
      - 8080:8080 # hhtp port
      - 18080:18080 # grpc port
      - 9325:9325 # metrics port
    volumes:
        - path/to/volume_data/local:path/to/volume_data/container
    command: 'volume -mserver="master_server_addr:port" -ip="local_or_publick_addr" -port=8080 -port.grpc=18080 -ip.bind=0.0.0.0 -port=8080  -metricsPort=9325 -dir=/path/to/volume_data/container -dataCenter=dc -rack=rc' # важно указывать ip.bind=0.0.0.0
```
Вызовете ```sudo docker-compose up``` или ```sudo docker-compose up -d``` для старта `volume` сервиса, ```sudo docker-compose down``` для остановки.
## Docker
Добавление `volume` сервиса в имеющуюся инфраструктуру в _docker_ на этой же машине 
```
	docker run --name weed_d_v -d -p 8081:8081 -p 18081:18081 \
	--network="weed_network" \
  	-v local_data:/data \
	chrislusf/seaweedfs volume \
	-mserver="master:9333" \
	-ip="weed_d_v" \
	-ip.bind="0.0.0.0" \
	-port=8081 \
	-port.grpc=18081 \
	-dataCenter="dc1" \
	-rack="rc1" \
	-dir="/data" 
```
Добавление в инфраструктуру на другой машине
```
	docker run --name weed_d_v -d -p 8081:8081 -p 18081:18081 \
  	-v local_data:/data \
	chrislusf/seaweedfs volume \
	-mserver="master_addr:9333" \
	-ip="publick_addr" \
	-ip.bind="0.0.0.0" \
	-port=8081 \
	-port.grpc=18081 \
	-dataCenter="dc1" \
	-rack="rc1" \
	-dir="/data" 
```


