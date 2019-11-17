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

## ingress для nginx контроллера
```
kubectl apply -f nginx-ingress.yaml
kubectl get ing
```

```
curl my.s000000.edu.slurm.io
```

> Правим в файле host-ingress.yaml 's000000' на свой номер студента
```
kubectl apply -f host-ingress.yaml
kubectl get ing
```

> работает
```
curl my.s000000.edu.slurm.io
```
> отдает 404
```
curl notmy.s000000.edu.slurm.io
```

# Переходим к установке cert-manager

```
cd cert-manager
cat instruction.md
```
