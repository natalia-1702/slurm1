## Push

В этой части к имеющемся шагам добавляем image push в registyr. В качестве registry используется gitlab registry. Push выполняется только в случае, если предидущие шаги закончились успешно.

**1. Подготавливаем ci/cd**

Для этого скопируем `.gitlab-ci.yml` в проект `xpaste` выполнив команду:

```bash
$ cp ~/slurm/practice/10.ci-cd/3.push/.gitlab-ci.yml ~/xpaste/
```
Добавим файл `.gitlab-ci.yml` в git и запушим изменения выполнив следующие команды:

```bash
$ cd ~/xpaste
$ git add .
$ git commit -am "Init ci/cd. Add push stage."
$ git push
```

**2. Проеврка результата**

Для проверки результата необходимо перейти в gitlab в раздел `ci/cd -> piplines` форка проекта xpaste. 
Можно воспользоваться прямой ссылкой: `https://gitlab.slurm.io/sXXXXXX/xpaste/pipelines`. `sXXXXXX` необходимо заменить на номер своего студента.

В результате все job должны закончиться без ошибок.

