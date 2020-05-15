## Устанавливаем ceph CSI driver для CephFS

```
cd /srv/ceph

helm inspect values ceph-csi/ceph-csi-cephfs --version 2.1.0 >cephfs.yml
```

> Заполняем переменные в cephfs.yml

> Выполняем на node-1, чтобы узнать необходимые параметры:

```bash
#  - clusterID: "<cluster-id>"

ceph fsid
```
```bash
#     monitors:
#       - "<MONValue1>"
#       - "<MONValue2>"

ceph mon dump
```

> Правим файл cephfs.yml
> Заносим свои значение clusterID, и адреса мониторов.
> Включаем создание политик PSP и увеличиваем таймаут на создание дисков

> !!! Внимание, адреса мониторов указываются в простой форме адреc:порт, потому что они передаются в модуль ядра для монтирования cephfs на узел.
> !!! А модуль ядра еще не умеет работать с протоколом мониторов v2

> Список изменений в файле cephfs.yml. Опции в разделах nodeplugin и provisioner уже есть в файле, их надо исправить так как показано ниже.

```
csiConfig:
  - clusterID: "bcd0d202-fba8-4352-b25d-75c89258d5ab"
    monitors:
      - "172.18.8.5:6789"
      - "172.18.8.6:6789"
      - "172.18.8.7:6789"

nodeplugin:
  podSecurityPolicy:
    enabled: true

provisioner:
  replicaCount: 1
  timeout: 260s
  podSecurityPolicy:
    enabled: true
```

> При необходимости можно свериться с файлом cephfs/cephfs-values-example.yml

> Устанавливаем чарт

```
kubectl create ns ceph-csi-cephfs
helm upgrade -i ceph-csi-cephfs ceph-csi/ceph-csi-cephfs -f cephfs.yml -n ceph-csi-cephfs
```

> Заходим в каталог с практикой и далее переходим в каталог cephfs
> Правим манифесты:

>
> Провизионер для CephFS создает отдельных пользователей для каждого pv, поэтому ему нужны права администратора в кластере ceph
> Возвращаемся на node-1

> Посмотреть ключ доступа для пользователя admin

```bash
ceph auth get-key client.admin
```

> вывод:
AQCO9NJbhYipKRAAMqZsnqqS/T8OYQX20xIa9A==

> Выполняем на master-1, подставляем значение ключа в манифест секрета
> Если секрет был уже создан ранее и вы хотите его изменить, самый простой способ - удалить и создать заново

```bash
cd ~/slurm/practice/7.ceph/cephfs
vi secret.yaml
# Заносим значение ключа в 
# adminKey: AQBRYK1eo++dHBAATnZzl8MogwwqP/7KEnuYyw==

# Создаем секрет

kubectl apply -f secret.yaml
```

>
> Создаем storage class
> Выполняем на node-1

```bash
# Получаем id кластера ceph

ceph fsid
```

> Выполняем на master-1

```
# Заносим clusterid в storageclass.yaml
vi storageclass.yaml
# clusterID: bcd0d202-fba8-4352-b25d-75c89258d5ab

kubectl apply -f storageclass.yaml
```

> Проверяем как работает.
> Создаем pvc, и проверяем статус и наличие pv

```bash
kubectl apply -f pvc.yaml

kubectl get pvc
kubectl get pv
```

> Проверяем создание каталога в cephfs
> Выполняем на node-1

```bash
# Точка монтирования
mkdir -p /mnt/cephfs

# Создаем файл с ключом администратора
ceph auth get-key client.admin >/etc/ceph/secret.key

# Добавляем запись в /etc/fstab
# !! Изменяем ip адрес на адрес узла node-1
echo "172.<xx>.<yyy>.6:6789:/ /mnt/cephfs ceph name=admin,secretfile=/etc/ceph/secret.key,noatime,_netdev    0       2">>/etc/fstab

mount /mnt/cephfs
```

> Идем в каталог /mnt/cephfs и смотрим что там есть

## Деплоим приложение fileshare

> Меняем в каталоге fileshare манифесты:

```bash
cd ~/slurm/practice/7.ceph/fileshare
```
в ingress.yaml host: номер своего студента

```bash
kubectl apply -f .
```

> Смотрим, что все поднялось
```bash
kubectl get pod
```
> Если какие то проблемы - то смотрим describe пода
```bash
kubectl describe pod названиепода
```

Пробуем загрузить файл
```
curl -i fileshare.s<номер своего логина>.edu.slurm.io/files/ -T configmap.yaml
```

Идем на node-1 и в каталоге /mnt/cephfs/volumes/csi/csi-vol-.... убеждаемся, что там появился файл configmap.yaml


> Пробуем как работает resize
> Изменяем размер тома в манифесте pvc-cephfs.yaml

```bash
vi pvc-cephfs.yaml

resources:
  requests:
    storage: 20Mi
```

> Ресайз отработает, но с фактическим изменением квот могут быть проблемы

> Проверяем на node-1

```bash
yum install -y attr

getfattr -n ceph.quota.max_bytes <каталог-с-данными>
```

## Возвращаемся в advanced abstraction, statefulset.
