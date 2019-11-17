## Все работы проводим с первого мастера.

### установка Helm

```bash
yum install helm -y

kubectl create serviceaccount tiller --namespace kube-system
kubectl create clusterrolebinding tiller --clusterrole=cluster-admin --serviceaccount=kube-system:tiller

helm init --service-account tiller --history-max 10
```

### Helm Cheatsheet

Поиск чартов

```bash
helm search
```

Получение дефолтных values

```bash
helm inspect values repo/chart > values.yaml
```

Установка чарта в кластер
```bash
helm upgrade --install release-name repo/chart [--atomic] [--namespace namespace]
```
