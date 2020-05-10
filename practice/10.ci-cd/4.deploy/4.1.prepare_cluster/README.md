### Подготовка кластера

**1. Создаем ns и RBAC**

Для этого запускаем скрипт setup.sh, перед запуском `<номер своего логина> меняем на свой номер студента!!`. 
```bash
$ bash ~/slurm/practice/10.ci-cd/4.deploy/4.1.prepare_cluster/setup.sh s<номер своего логина>-xpaste production
```
В конце своего выполнения скрипт выдаст нам токен, который необходимо сохранить.

**2. Создание variables в gitlab**

Для доступа из Gitlab в Kubernetes нам необходимо добавить переменную в Gitlab в которой будет содержаться токен с предидущего шага.

* Перходим в gitlab 
Для этого открываем в браузере свой форк xpaste. `<номер своего логина> меняем на свой номер студента`:
```bash
https://gitlab.slurm.io/s<номер своего логина>/xpaste
```
* Добавляем переменную
Для этого в левом меню находим `Settings -> CI/CD -> Variables` и нажимаем Expand. В поле `Name` вводим имя переменной:
```bash
K8S_CI_TOKEN
```
В поле `value` вводим скопированный токен из вывода команды setup.sh(пункт 1) и нажимаем `Save variables`.

**3. Создаем token для доступа в registry**

Для этого переходим в раздел `Settings -> Repository -> Deploy tokens` и нажимаем Expand.

В поле `Name` вводим
```bash
k8s-pull-token
```
Cтавим галочку рядом с `read_registry`. Все остальные поля оставляем пустыми. Нажимаем `Create deploy token`.

```!!НЕ ЗАКРЫВАЕМ ОКНО БРАУЗЕРА!!```

**4. Создаем secret kubernetes**

Созздаем secret который будет использоваться для image pull. Для этого возвращаемся на первый master и выполняем команду:
```bash
kubectl create secret docker-registry xpaste-gitlab-registry --docker-server registry.slurm.io --docker-email 'student@slurm.io' --docker-username '<первая строчка из окна создания токена в gitlab>' --docker-password '<вторая строчка из окна создания токена в gitlab>' --namespace s<номер своего логина>-xpaste-production
```
Соответсвенно подставляя на место `<>` нужные параметры которые получили на `шаге 3`.

**5. Установка PostgreSQL**

Postgres устанавливаем в kubernetes кластер, делается это исключительно в учебных целях. Для установки будем использована утилита helm3. Для установки необходимо выполнить следующую команду, заменив `<номер своего логина>` на номер своего студента:
```bash
$ cd ~/slurm/practice/10.ci-cd/4.deploy/4.1.prepare_cluster/
$ helm3 install postgresql postgresql --namespace s<номер своего логина>-xpaste-production --atomic --timeout 120s
```

**6. Создание секрета для приложения**

Создаем секрет из которого при деплое будут взяты значения для env для таких переменных как доступы к БД и секретный ключ. Для этого выполним команду:
```bash
kubectl create secret generic slurm-xpaste --from-literal secret-key-base=xxxxxxxxxxxxxxxxxxxxxxxxx --from-literal db-user=postgres --from-literal db-password=postgres --namespace s<номер своего логина>-xpaste-production
```

`secret-key-base xxxxxxxxxxxxx` это не плэйсхолдер. Можно так и оставит.
