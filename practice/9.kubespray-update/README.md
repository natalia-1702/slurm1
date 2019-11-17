# Всё просто

>исправляем значение переменной kube_version: в файле
>group_vars/k8s-cluster/k8s-cluster.yml

```
kube_version: v1.15.3
```

>Исправляем путь к инвентарю в скрипте _upgrade_cluster.sh (000 меняем на номер студента)
>inventory/s000/inventory.ini

>и запускаем скрипт на обновление кластера

```
sh _upgrade_cluster.sh <свой логин>
```
