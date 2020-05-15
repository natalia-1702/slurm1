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
