### Pod

**1. Создаем pod**

Для этого выполним команду:
```bash
kubectl apply -f ~/slurm/practice/1.kube-basics-lecture/1.pod/pod.yaml
```
Проверим результат для этого выполним команду:
```bash
kubectl get pod
```
Результат должен быть примерно следующим:
```bash
NAME      READY     STATUS              RESTARTS   AGE
my-pod    0/1       ContainerCreating   0          2s
```
Через какое-то время pod должен перейти в состояние `Running`
и вывод команды `kubectl get po` станет таким:
```bash
NAME      READY     STATUS    RESTARTS   AGE
my-pod    1/1       Running   0          59s
```
**2. Скейлим приложение**

Открываем файл pod.yaml на редактирование:
```bash
vim pod.yaml
```
И заменяем там строку:
```diff
-  name: my-pod
+  name: my-pod-1
```
Сохраняем и выходим. Для vim нужно нажать `:wq<Enter>`

Пирменяем изменения, для этого выполним команду:
```bash
kubectl apply -f ~/slurm/practice/1.kube-basics-lecture/1.pod/pod.yaml
```
Проверяем результат, для этого выполним команду:
```bash
kubectl get pod
```
Результат должен быть примерно следующим:
```bash
NAME      READY     STATUS    RESTARTS   AGE
my-pod    1/1       Running   0          10m
my-pod-1  1/1       Running   0          59s
```
**3. Обновляем версию image**

Обновляем версию image в pod my-pod, для этого выполним команду:
```bash
kubectl edit pod my-pod
```
И заменяем там строку:
```diff
-  - image: nginx:1.12
+  - image: nginx:1.13
```
Проверяем результа, для этого выполним команду:
```bash
kubectl describe po my-pod
```

В результате должны присутствовать строки:
```bash
Containers:
  nginx:
    Container ID:   
    Image:          nginx:1.13
```

**4. Чистим за собой кластер**

Для этого выполним команду:
```bash
kubectl delete pod --all
```
