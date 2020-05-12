# Переходим в директорию с практикой 5.ingress в папку 'cert-manager':
```
cd slurm/practice/5.ingress/cert-manager
```

# Устанавливаем cert-manager, выпускаем тестовый сертификат
```
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.14.2/cert-manager.crds.yaml

kubectl create namespace cert-manager

helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v0.14.2
```

## Проверяем работу, выпустив самоподписанный сертификат
```
kubectl apply -f test-resources.yaml
```

Если возникает ошибка `Error from server (InternalError)`, то применяем манифест еще раз.
## Убеждаемся, что сертификат создан

> сертификат
```
kubectl -n cert-manager-test describe certificate selfsigned-cert
kubectl -n cert-manager-test get certificate selfsigned-cert -o yaml
```

> Секрет с содержимым сертификата
```
kubectl -n cert-manager-test describe secret selfsigned-cert-tls
kubectl -n cert-manager-test get secret selfsigned-cert-tls -o yaml
```

## Каким образом сертификат был выпущен - issuer
```
kubectl -n cert-manager-test get issuer
```

# Удаляем тестовый ns
```
kubectl delete ns cert-manager-test
```

# Создаем secret для авторизации в LE
```
kubectl create secret generic stage-issuer-account-key --from-file=./tls.key --namespace=cert-manager
```

# Проверяем что secret создался
```
kubectl describe secrets -n cert-manager stage-issuer-account-key
```

# Создаем выпускальщик сертификатов
```
kubectl apply -f clusterissuer-stage.yaml
```

# Проверяем что наш clusterissuer создался:
```
kubectl get clusterissuers letsencrypt -o yaml
```

# Добавляем в ingress информацию о TLS
> Правим в файле tls-ingress.yaml 'sXXXXXX' на свой номер студента
```
kubectl apply -f tls-ingress.yaml -n default
```

## посмотрели на сертификат
```
kubectl get certificate my-tls -o yaml
```

## посмотрели на секрет
```
kubectl get secret my-tls -o yaml
```

## Зайдем в браузер по адресу https://my.sXXXXXX.edu.slurm.io и убедимся что сертификат от issuer: CN=Fake LE Intermediate X1. Не забываем править sXXXXXX на свой номер студента

# Удаляем cert-manager

```
helm delete cert-manager --namespace cert-manager
```

# Вернулись назад
```
cd ..
```

# Удаляем все созданное ранее

```
kubectl delete -f app
kubectl get ing
kubectl delete ing my-ingress-nginx

kubectl get svc
kubectl delete svc my-service
kubectl delete svc my-service-np

kubectl delete ns cert-manager
```
