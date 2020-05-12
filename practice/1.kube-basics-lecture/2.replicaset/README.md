### ReplicaSet

**1. Создаем replicaset**

Для этого выполним команду:
```bash
$ kubectl apply -f ~/slurm/practice/1.kube-basics-lecture/2.replicaset/replicaset.yaml
```
Проверим результат для этого выполним команду:
```bash
$ kubectl get pod
```
Результат должен быть примерно следующим:
```bash
NAME                  READY     STATUS              RESTARTS   AGE
my-replicaset-pbtdm   0/1       ContainerCreating   0          2s
my-replicaset-z7rwm   0/1       ContainerCreating   0          2s
```

**2. Скейлим replicaset**

Для этого выполним команду:
```bash
$ kubectl scale replicaset my-replicaset --replicas 3
```
Проверим результат для этого выполним команду:
```bash
$ kubectl get pod
```
Результат должен быть примерно следующим:
```bash
NAME                  READY     STATUS              RESTARTS   AGE
my-replicaset-pbtdm   1/1       Running             0          2m
my-replicaset-szqgz   0/1       ContainerCreating   0          1s
my-replicaset-z7rwm   1/1       Running             0          2m
```
**3. Удаляем один из подов**

Для этого выполним команду подставив имя своего pod(``можно воспользоваться автоподстановкой по TAB``):
```bash
$ kubectl delete pod my-replicaset-pbtdm
```
Проверим результат для этого выполним команду:
```bash
$ kubectl get pod
```
Результат должен быть примерно следующим:
```bash
NAME                  READY     STATUS              RESTARTS   AGE
my-replicaset-55qdj   0/1       ContainerCreating   0          1s
my-replicaset-pbtdm   1/1       Running             0          4m
my-replicaset-szqgz   1/1       Running             0          2m
my-replicaset-z7rwm   0/1       Terminating         0          4m
```
**4. Добавляем в репликасет лишний под**

Открываем файл `~/slurm/practice/1.kube-basics-lecture/2.replicaset/pod.yaml`
И в него после metadata: на следующей строке добавляем
```yaml
  labels:
    app: my-app
```
В итоге должно получиться:
```yaml
.............
kind: Pod
metadata:
  name: my-pod
  labels:
    app: my-app
spec:
.............
```
Создаем дополнительный pod, для этого выполним команду:
```bash
$ kubectl apply -f ~/slurm/practice/1.kube-basics-lecture/2.replicaset/pod.yaml
```
Проверяем результа для этого выполним команду:
```bash
$ kubectl get pod
```
Результат должен быть примерно следующим:
```bash
NAME                  READY     STATUS        RESTARTS   AGE
my-pod                0/1       Terminating   0          1s
my-replicaset-55qdj   1/1       Running       0          3m
my-replicaset-pbtdm   1/1       Running       0          8m
my-replicaset-szqgz   1/1       Running       0          6m
```
**5. Обновляем версию image для container**

Для этого выоплним команду:
```bash
$ kubectl set image replicaset my-replicaset nginx=nginx:1.13
```
Проверяем результа для этого выполним команду:
```bash
$ kubectl get pod
```
Результат должен быть примерно следующим:
```bash
NAME                  READY     STATUS        RESTARTS   AGE
my-replicaset-55qdj   1/1       Running       0          3m
my-replicaset-pbtdm   1/1       Running       0          8m
my-replicaset-szqgz   1/1       Running       0          6m
```
И проверяем сам replicaset для этого выполним команду:
```bash
$ kubectl describe replicaset my-replicaset
```
В результате находим строку Image и видим:
```bash
  Containers:
   nginx:
    Image:        nginx:1.13
```
Проверяем версию image в pod для этого выполним команду, подставив имя своего pod(``можно воспользоваться автоподстановкой по TAB``):
```bash
$ kubectl describe pod my-replicaset-55qdj
```
Видим что версия имаджа в поде не изменилась:
```bash
  Containers:
   nginx:
    Image:        nginx:1.12
```
Помогаем поду, для этого выполним команду, подставив имя своего pod(``можно воспользоваться автоподстановкой по TAB``):
```bash
$ kubectl delete po my-replicaset-55qdj
```
Проверяем результа для этого выполним команду:
```bash
$ kubectl get pod
```
Результат должен быть примерно следующим:
```bash
NAME                  READY     STATUS              RESTARTS   AGE
my-replicaset-55qdj   0/1       Terminating         0          11m
my-replicaset-cwjlf   0/1       ContainerCreating   0          1s
my-replicaset-pbtdm   1/1       Running             0          16m
my-replicaset-szqgz   1/1       Running             0          14m
```
Проверяем версию имаджа в новом pod для этого выполним команду, подставив имя своего pod(``можно воспользоваться автоподстановкой по TAB``):
```bash
$ kubectl describe pod my-replicaset-cwjlf
```
Результат должен быть примерно следующим:
```bash
    Image:          nginx:1.13
```
**6. Чистим за собой кластер**

Для этого выполним команду:
```bash
$ kubectl delete replicaset --all
```