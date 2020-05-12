# Переходим в директорию с практикой 5.ingress:

```
cd slurm/practice/5.ingress
```

# Деплоим приложение

```
kubectl apply -f app
```

# Деплоим разные типы сервисов
## ClusterIP
```
kubectl apply -f clusterip.yaml
```

## Nodeport
```
kubectl apply -f nodeport.yaml
```

# Ingress для nginx контроллера
```
kubectl apply -f nginx-ingress.yaml
kubectl get ing
```

```
curl my.sXXXXXX.edu.slurm.io <- Подставляем свой номер логина
```

Правим в файле host-ingress.yaml 'sXXXXXX' на свой номер студента

```
kubectl apply -f host-ingress.yaml
kubectl get ing
```

Проверяем
```
curl my.sXXXXXX.edu.slurm.io
```
Запрос на другой домен отдает 404
```
curl notmy.sXXXXXX.edu.slurm.io
```

# Переходим к установке cert-manager

```
cd cert-manager
cat README.md
```
