# Rebase

`rebase` создаёт более чистую историю коммитов, чем `merge`. Рассмотрим на примере команды `pull`.

Переключитесь на новую ветку, начав её от первого коммита в репозитории. Подставьте свой `hash`,
посмотрев его в `git log`.

``` 
git checkout -b feature/rebase 91d0e04
```

Создайте и добавьте новый файл.

``` 
touch empty_file.txt
git add .
git commit -m "Добавил пустой файл про запас"
```

```
* 003bf78 (HEAD -> feature/rebase) Добавил пустой файл про запас
* 91d0e04 Добавлен README.md
```

Выполните `git pull origin master` и посмотрите историю
коммитов `git log --graph --oneline --decorate`.

```
*   23e23e7 (HEAD -> feature/rebase) Merge branch 'master' of github.com:username/git_intro into feature/rebase
|\  
| *   003bf78 (origin/master) Merge pull request #2 from username/feature/branches
| |\  
| | * 16e30f8 (origin/feature/branches) Добавлена информация о ветвлении
| |/  
| * add34c9 Добавлена информация о команде reset
| * 2b6a10d Добавлена информация о команде log
| * 689a900 Добавлена информация о команде commit
* | eec8dae Добавил пустой файл про запас
|/  
* 91d0e04 Добавлен README.md
```

Мастер-ветка вмёржилась в `feature/rebase`, создан автоматический коммит о слиянии.

Теперь вернитесь в исходное состояние ветки.

```
git reset --soft 6ca257d
git stash
git reset --hard 91d0e04 
git stash pop
git commit -m "Добавил пустой файл про запас"
```

Выполните команду `git pull --rebase origin master` и сравните историю.

```
* bf9d8da (HEAD -> feature/rebase) Добавил пустой файл про запас
*   6ca257d (origin/master) Merge pull request #2 from username/feature/branches
|\  
| * 16e30f8 (origin/feature/branches) Добавлена информация о ветвлении
|/  
* add34c9 Добавлена информация о команде reset
* 2b6a10d Добавлена информация о команде log
* 689a900 Добавлена информация о команде commit
* 91d0e04 Добавлен README.md
```

В отличие от `merge`, который выполняет слияние веток в отдельном коммите, `rebase` перемещает ваш
коммит в самый верх ветки. Подробнее о `rebase` можете почитать
в [статье на Habr](https://habr.com/ru/post/432420/).

## Rebase с изменением коммитов

Ещё одно преимущество ребейза — одной командой все коммиты схлопываются в один. История становится
более чистой, а коммиты с мелкими фиксами или исправлениями комментариев ревьюверов отсутствуют.

Для примера создайте пачку мелких изменений.

```  
touch counter.md && git add . && git commit -m "init counter"
for i in {1..5}; do echo $i >> counter.md; git add . ; git commit -m $i; done
```

Загляните в лог.

```
a24359e (HEAD) 5
4139f15 4
16517e6 3
a64a3fa 2
fa732c5 1
8829b18 init counter
```

Запустите `rebase` в интерактивном режиме редактирования истории.

```
git rebase [-i | --interactive] HEAD~6
```

Вы увидите список коммитов и инструкцию по применению `rebase`.

```
pick 8829b18 init counter
pick fa732c5 1
pick a64a3fa 2
pick 16517e6 3
pick 4139f15 4
pick a24359e 5

# Rebase 0eba08d..a24359e onto 0eba08d (6 commands)
#
# Commands:
# p, pick <commit> = use commit
# r, reword <commit> = use commit, but edit the commit message
# e, edit <commit> = use commit, but stop for amending
# s, squash <commit> = use commit, but meld into previous commit
# f, fixup <commit> = like "squash", but discard this commit's log message
# x, exec <command> = run command (the rest of the line) using shell
# d, drop <commit> = remove commit
# l, label <label> = label current HEAD with a name
# t, reset <label> = reset HEAD to a label
# m, merge [-C <commit> | -c <commit>] <label> [# <oneline>]
```

Ваша задача — указать для каждого коммита необходимое изменение. Например, чтобы соединить коммиты
со второго по пятый, нужно объявить их как `squash`.

```
pick 8829b18 init counter
pick fa732c5 1
squash a64a3fa 2
squash 16517e6 3
squash 4139f15 4
squash a24359e 5`
``` 

После нажатия `ctrl+c` (перейти в режим ввода команд в [vim](https://www.vim.org/)) 
и ввода `:x` (выйти и сохранить) Git предложит ввести сообщение для нового коммита. 
Очистите поле, последовательно нажав `d` (удалить) и `G` (перейти в конец файла), 
перейдите в режим ввода кнопкой `i` и введите нужный текст.

Проверьте историю после завершения `rebase`. Она стала гораздо чище!

```
3f446fb (HEAD -> feature/rebase) add numbers
8829b18 init counter
```

## Будьте аккуратны!

Если вы меняете историю коммитов, используя `rebase`, `ammend` или `reset` в ветке, которая уже есть
в `origin`, вы получите сообщение об ошибке. Git не сможет автоматически проассоциировать имеющиеся
в `origin` коммиты с вашей изменённой версией. Чтобы решить проблему, вам придётся обновить ветку
«силой», использовав `push --force`. К этому можно прибегать, только если вы работаете один в своей
ветке, иначе после смены хэша коммитов конфликты получат все остальные разработчики.

## Как откатить rebase, если его сделали не от нужной ветки

Попробуйте решить следующую ситуацию: разработчики создали ветку и закоммитили в неё кучу изменений.
Затем сделали ребейз от ветки `feature/1`. Прибегает тимлид и говорит, что надо было ребейзиться от
ветки `feature/2`. Как быть?

Один из ключей к решению этой проблемы — команда `git reflog`.

## Недостатки rebase

Из-за того, что ребейз меняет историю коммитов, можно потерять контекст, в котором их написали
разработчики. Ещё теряется информация о слияниях веток: она нужна для определения причины ошибки.
Помните об этом.
