## Ставим ceph с помощью ceph-ansible
> Все работы проводим под рутом на master-1
> С sbox заходим на master-1

```bash
ssh master-1.s<номер вашего логина>
sudo -s
```
> Клонируем сценарий автоматизированной установки ceph в каталог /srv
```bash
cd /srv
git clone git@gitlab.slurm.io:slurm/ceph-nautilus.git
```
> Устанавливаем ansible и зависимости
```bash
cd ceph-nautilus
```

> Запускаем сценарий скриптом, который автоматически поправит инвентарь

```bash
sh  _deploy_cluster.sh
```
> Следим за выводом на экран. Ждем отчета об успешном завершении

> Если все хорошо, последний таск сценария покажет в выводе:
`health: HEALTH_OK`

> Если сообщение не HEALTH_OK - есть вероятность, что
не все компоненты успели запустится, проверим позже на node-1.

> Далее будем работать на двух серверах
> node-1 и master-1
> Все комадны `kubectl` выполняются на master-1
> Команды `ceph` и `rbd` выполняются на node-1

> С sbox заходим на node-1

```bash
ssh node-1.s<номер вашего логина>
sudo -s
```
> Проверяем что цеф живой

```bash
ceph health
ceph -s
```

## Создаем пул в цефе для RBD дисков
> делаем на node-1

```bash
ceph osd pool create kube 32
ceph osd pool application enable kube kubernetes
```

## Устанавливаем ceph CSI driver для RBD


> Добавляем репозиторий с чартом, получаем набор переменных чарта ceph-csi-rbd

```
helm repo add ceph-csi https://ceph.github.io/csi-charts

mkdir -p /srv/ceph
cd /srv/ceph

helm inspect values ceph-csi/ceph-csi-rbd --version 2.1.0 >cephrbd.yml
```

> Заполняем переменные в cephrbd.yml

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

> Правим файл cephrbd.yml
> Заносим свои значение clusterID, и адреса мониторов.
> включаем создание политик PSP, и увеличиваем таймаут на создание дисков
> Список изменений в файле cephrbd.yml. Опции в разделах nodeplugin и provisioner уже есть в файле, их надо исправить так как показано ниже.

```
csiConfig:
  - clusterID: "bcd0d202-fba8-4352-b25d-75c89258d5ab"
    monitors:
      - "v2:172.18.8.5:3300/0,v1:172.18.8.5:6789/0"
      - "v2:172.18.8.6:3300/0,v1:172.18.8.6:6789/0"
      - "v2:172.18.8.7:3300/0,v1:172.18.8.7:6789/0"

nodeplugin:
  podSecurityPolicy:
    enabled: true

provisioner:
  replicaCount: 1
  timeout: 260s
  podSecurityPolicy:
    enabled: true
```

> При необходимости можно сверится с файлом rbd/cephrbd-values-example.yml

> Устанавливаем чарт

```
kubectl create ns ceph-csi-rbd
helm upgrade -i ceph-csi-rbd ceph-csi/ceph-csi-rbd -f cephrbd.yml -n ceph-csi-rbd
```

> Заходим в каталог с практикой и далее переходим в каталог rbd
> Правим манифесты:

>
> Создаем пользователя в ceph, с правами записи в пул kube
> Возвращаемся на node-1

```bash
ceph auth get-or-create client.rbdkube mon 'allow r, allow command "osd blacklist"' osd 'allow rwx pool=kube'
```

> Посмотреть ключ доступа для пользователя ceph

```bash
ceph auth get-key client.rbdkube
```

> вывод:
AQCO9NJbhYipKRAAMqZsnqqS/T8OYQX20xIa9A==

> выполняем на master-1, подставляем значение ключа в манифест секрета секрета
> Если секрет был уже создан ранее и вы хотите его изменить, самый простой способ - удалить и создать заново

```bash
vi secret.yaml
# Заносим значение ключа в 
# userKey: AQBRYK1eo++dHBAATnZzl8MogwwqP/7KEnuYyw==

# Создаем секрет

kubectl apply -f secret.yaml
```

>
> Создаем storage class
> Выполняем на node-1

```bash
# Получам id кластера ceph

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
> Создаем pvc, и проверем статус и наличие pv

```bash
kubectl apply -f pvc.yaml

kubectl get pvc
kubectl get pv
```

> Проверяем создание тома в цеф
> Выполняем на node-1

```bash
# список томов в пуле kube
rbd ls -p kube

# информация о созданном томе
rbd -p kube info csi-vol-eb3d257d-8c6c-11ea-bff5-6235e7640653
```

> Пробуем как работает resize
> Изменяем размер тома в манифесте pvc.yaml

```bash
vi pvc.yaml

resources:
  requests:
    storage: 2Gi
```

> Немножко ждем и проверяем размер тома в ceph, pv, pvc

```bash
rbd -p kube info csi-vol-eb3d257d-8c6c-11ea-bff5-6235e7640653

kubectl get pv
kubectl get pvc
```

> У pvc размер не изменился, и если посмтреть его манифест в yaml

```bash
kubectl get pvc rbd-pvc -o yaml
```

> То увидим сообщение message: Waiting for user to (re-)start a pod to finish file system resize of volume on node.
> type: FileSystemResizePending
> Необходимо смонтировать том, для того чтобы увеличить на нем файловую систему. А у нас этот PVC/PV не используется никаким подом.
> Создаем под, который использует этот PVC/PV, и смотри на размер, указанный в pvc

```bash
kubectl apply -f pod.yaml

kubectl get pvc
```

----------------------------
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
> включаем создание политик PSP, и увеличиваем таймаут на создание дисков

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

> При необходимости можно сверится с файлом cephfs/cephfs-values-example.yml

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

> выполняем на master-1, подставляем значение ключа в манифест секрета секрета
> Если секрет был уже создан ранее и вы хотите его изменить, самый простой способ - удалить и создать заново

```bash
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
# Получам id кластера ceph

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
> Создаем pvc, и проверем статус и наличие pv

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

# Добавляем запсиь в /etc/fstab
# !! Изменяем ip адрес на адрес узла node-1
echo "172.<xx>.<yyy>.6:6789:/ /mnt/cephfs ceph name=admin,secretfile=/etc/ceph/secret.key,noatime,_netdev    0       2">>/etc/fstab

mount /mnt/cephfs
```

> Идем в каталог /mnt/cephfs и смотрим что там есть

## Деплоим приложение fileshare

> Меняем в каталоге fileshare манифесты:

```bash
cd fileshare
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

Идем на node-1 и в каталоге /mnt/cephfs/pvc-........ убеждаемся, что там появился файл configmap.yaml


> Пробуем как работает resize
> Изменяем размер тома в манифесте pvc.yaml

```bash
vi pvc.yaml

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
