## Ставим ceph с помощью ceph-ansible
> Все работы проводим под рутом на master-1
> С sbox заходим на master-1

```bash
ssh master-1.s000001
sudo -s
```
> Клонируем сценарий автоматизированной установки ceph в каталог /srv
```bash
cd /srv
git clone git@gitlab.slurm.io:slurm/ceph.git
```
> Устанавливаем ansible и зависимости
```bash
cd ceph
pip install -r requirements.txt
```

> Запускаем сценарий скриптом, который автоматически поправит инвентарь

```bash
sh  _deploy_cluster.sh
```
> Следим за выводом на экран. Ждем отчета об успешном завершении

> Если все хорошо, последний таск сценария покажет в выводе:
`health: HEALTH_OK`

> Далее будем работать на двух серверах
> node-1 и master-1
> С sbox заходим на node-1

```bash
ssh node-1.s000001
sudo -s
```
> Проверяем что цеф живой

```bash
ceph health
```

## Создаем пул в цефе для RBD дисков
> делаем на node-1

```bash
ceph osd pool create kube 32
ceph osd pool application enable kube kubernetes
```
> создаем секрет с ключом админа в кубе
> смотрим ключ админа в цеф
> node-1
```bash
ceph auth get-key client.admin
```

> вывод:
 AQB8x9Jbk8RFKhAAfNVrIka3hlktspJUV9tnlw==

> выполняем на master-1, подставляем значение ключа в создание секрета
> Если секрет был уже создан ранее и вы хотите его изменить, самый простой способ - удалить и создать заново

```bash
# kubectl delete secret ceph-secret --namespace=kube-system
kubectl create secret generic ceph-secret --type="kubernetes.io/rbd" --from-literal=key='xxxxxxxxxxxxxxxxxxxx==' --namespace=kube-system
```

>
> Создаем пользователя в ceph, с правами записи в пул kube
> Возвращаемся на node-1

```bash
ceph auth get-or-create client.user mon 'allow r, allow command "osd blacklist"' osd 'allow rwx pool=kube'
```

```bash
ceph auth get-key client.user
```

> вывод:
AQCO9NJbhYipKRAAMqZsnqqS/T8OYQX20xIa9A==

> выполняем на master-1, подставляем значение ключа в создание секрета
> Если секрет был уже создан ранее и вы хотите его изменить, самый простой способ - удалить и создать заново

```bash
# kubectl delete secret ceph-secret-user --namespace=default
kubectl create secret generic ceph-secret-user --type="kubernetes.io/rbd" --from-literal=key='yyyyyyyyyyyyyyyyyyyyyyyyyyy==' --namespace=default
```

> создаем StorageClass
>Редактируем файл sc.yml, в этом файле правим сеть у адресов мониторов
parameters:
  monitors: 172.xx.xxx.5:6789, 172.xx.xxx.6:6789, 172.xx.xxx.7:6789

>В поле userId: я уже указал имя пользователя user

> Команды с kubectl delete удаляют существующий объект, чтобы можно было создать новый
> Аналогично с изменением storageclass - удаляем и создаем заново
> sc.yml лежит в каталоге с практикой practice/7.ceph
> master-1

```bash
# kubectl delete sc kube
kubectl apply -f sc.yml
```

> Проверяем что создался sc
```bash
kubectl get sc
```
>
> Создаем pvc
> в файле pvc.yml лежит манифест для PersistenceVolumeClaim
> pvc.yml лежит в каталоге с практикой practice/7.ceph
> на master-1

```bash
kubectl apply -f pvc.yml --namespace=default
```
> команды диагностики:
> запускать на master-1

> посмотреть список созданных PersistentVolume
```bash
kubectl get pv
```
> посмотреть на ошибки создания тома
```bash
kubectl describe pvc prometheus-3gb --namespace=default
```

> Самая основная проблема была такая:
>в describe pvc было вот такое написано:
>  Warning    ProvisioningFailed  3s                    persistentvolume-controller  Failed to provision volume with StorageClass "kube": failed to get admin secret from ["kube-system"/"ceph-secret"]: failed to get secret from ["kube-system"/"ceph-secret"]

>Помогло удаление секрета админа и создание его заново.

> остальные ошибки были из-за проблем с указанием не тех ip адресов мониторам или не тех username для admin и user

## Создание пула для Cephfs не требуется. оба пула были созданы сценарием ceph-ansible

## Подключим том типа cephfs вручную

> Делаем на node-1
> Создаем каталог, в команде монтирования правим сеть у адресов мониторов
> в 172.xx.xxx.6

```bash
mkdir -p /mnt/cephfs
mount.ceph 172.21.0.6:/ /mnt/cephfs -o name=admin,secret=`ceph auth get-key client.admin`
mkdir -p /mnt/cephfs/data_path
```

## Создаем пользователя и секреты для доступа к cephfs
> Делаем на node-1

```bash
ceph auth get-or-create client.username mon 'allow r' mds 'allow r, allow rw path=/data_path' osd 'allow rw pool=cephfs_data' 
```

> команда вернет ключ доступа пользователя

> Делаем на master-1
```bash
kubectl create secret  generic cephfs-secret-username --from-literal=key='ключ доступа пользователя username' --namespace default
```

## Деплоим приложение fileshare

> Меняем в каталоге fileshare манифесты:

```bash
cd fileshare
```
в ingress.yml host: номер своего студента
в deployment.yml: в описании тома data типа cephfs - ставим ip адреса своих серверов mon ceph. (подставляем свою сеть)

```bash
kubectl apply -f .
```

> Смотрим, что все поднялось
```bash
kubectl get pod
```
> Если какие то проблемы - то смотрим describe пода
```bash
kubectl describe названиепода
```

Пробуем загрузить файл
```
curl -i fileshare.s<номер своего логина>.edu.slurm.io/files/ -T configmap.yaml
```

Идем на node-1 и в каталоге /mnt/cephfs/data_path убеждаемся, что там появился файл configmap.yaml


## Установка CephFS provisioner

> Добавляем репо sb с чартами, получаем переменные чарта и правим их

```bash
helm repo add sb https://charts.southbridge.ru 
helm inspect values sb/cephfs-provisioner > values.yaml

> также меняем сеть в адресах мониторов

vim values.yaml
```
<<<<<<<<<<<
```yaml
adminSecretName: ceph-secret-admin
monitors:
  - 172.21.0.5:6789
  - 172.21.0.6:6789
  - 172.21.0.7:6789
```

## Создаем секрет с ключом администратора ceph и ставим провизионер

> на node-1
```bash
ceph auth get-key client.admin
```
> на master-1

```bash
kubectl create secret generic ceph-secret-admin --namespace=kube-system  --from-literal=secret=XX
helm upgrade --install cephfs-provisioner sb/cephfs-provisioner -f values.yaml  --namespace=kube-system 
```
## Создаем pvc для cephfs
> pvc-cephfs.yml лежит в каталоге с практикой practice/7.ceph

```bash
kubectl apply -f pvc-cephfs.yml
```

## Проверяем, что pv был создан

```bash
kubectl get pv
```
## Самостоятельная задача. Надо исправить deployment для fileshare, чтобы том data использовал pvc fileshare, а не подключался к cephfs напрямую

> исправить, передеплоить и попробовать загрузить файл

```
curl -i fileshare.s<номер своего логина>.edu.slurm.io/files/ -T configmap.yaml
```

> Идем на node-1 и в каталоге /mnt/cephfs/k8s/default/fileshare убеждаемся, что там появился файл configmap.yaml

