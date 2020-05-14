### ВНИМАНИЕ!

**Все работы проводим на первом мастере!**

### установка Helm

```bash
yum install helm -y
```

### Подключаем repo и смотрим kube-ops-view

```
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
helm repo update

helm search hub kube-ops
helm show values stable/kube-ops-view > values.yaml

```

Правим `values.yaml`:

```
ingress:
  enabled: true
...
hostname: kube-ops.sXXXXXX.edu.slurm.io
...
rbac:
  create: true
...
```
где `sXXXXXX` - ваш номер студента

Устанавливаем kube-ops-view:

```
helm install ops-view stable/kube-ops-view --namespace kube-system -f values.yaml
```

Переходим в браузер в Инкогнито режим и заходим на `http://kube-ops.sXXXXXX.edu.slurm.io/`

Удаляем чарт:

```
helm delete ops-view --namespace kube-system
```

### Посмотрим, что внутри чарта:

```
helm pull stable/kube-ops-view

tar -zxvf kube-ops-view-1.1.4.tgz

cd kube-ops-view/
```

### Helm Cheatsheet

Поиск чартов

```
helm search hub
```

Получение дефолтных values

```
helm show values repo/chart > values.yaml
```

Установка чарта в кластер

```
helm install release-name repo/chart [--atomic] [--namespace namespace]
```

Локально отрендерить чарт

```
helm template /path/to/chart
```

### Если отстали по практике на каком-то месте, то смотрим ```summary_file.yaml```

---

**ДОМАШНЯЯ РАБОТА:**

- Перейти в папку `homework`
- Запустить deployment из файла `bad_deployment.yaml`
- Исправить все найденные ошибки и сделать так, чтобы все pod'ы были в состоянии `Running 1/1`

Ответ-шпаргалка находится в файле `bad_deployment.yaml_otvet`
