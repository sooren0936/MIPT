# Конфликты

Бывают ситуации, когда один и тот же файл разработчики одновременно изменили в разных ветках и при
слиянии этих веток происходят конфликты. Потренируемся их создавать и ликвидировать.

Сейчас ваша история выглядит примерно так:

```
32a3cd9 (HEAD -> master, origin/master) Добавлена информация о merge
6ca257d Merge pull request #2 from username/feature/branches
16e30f8 Добавлена информация о ветвлении
add34c9 Добавлена информация о команде reset
2b6a10d Добавлена информация о команде log
689a900 Добавлена информация о команде commit
91d0e04 Добавлен README.md
```

Переключитесь на ветку `yoda` и добавьте коммит.

```
git checkout -b yoda master
echo "Always more questions than answers there are." >> quotes.md
```

Теперь перейдите в ветку `human` и сделайте похожие изменения.

```
git checkout -b human master
echo "There are always more questions than answers." >> quotes.md
```

Попробуйте вмёржить ветку `yoda`.

```
git merge yoda
```

Git выведет сообщение о конфликте...

```
CONFLICT (add/add): Merge conflict in quotes.md
Auto-merging quotes.md
Automatic merge failed; fix conflicts and then commit the result.
```

...и добавит в файлы пометки, где именно это произошло.

``` 
cat quotes.md
```

```
<<<<<<< HEAD
There are always more questions than answers.
=======
Always more questions than answers there are.
>>>>>>> yoda
```

В файле появились непонятные дополнения:

- `<<<<<<< HEAD`;
- `=======`;
- `>>>>>>> feature/yoda`.

Эти строки можно назвать «разделителями конфликта». Строка `=======` — «центр» конфликта. Всё
содержимое между строкой `<<<<<<< HEAD` и `=======` находится в текущей мастер-ветке. На неё и
указывает ссылка HEAD. А всё между центром и строкой `>>>>>>> feature/yoda` — содержимое той ветки,
с которой происходит слияние.

Для решения подобных конфликтов пользуются командой `git mergetool` и запускают программу для
слияния. Если такой нет, попробуйте установить [Meld](https://meldmerge.org) или просто уберите из
файла все лишние строки, оставив только корректные изменения. Если вы работаете в Idea, вызовите из
меню инструмент слияния: VCS → Git → Resolve conflicts.

Попробуйте решить конфликты самостоятельно и закоммитьте изменения.

Полюбуйтесь на ваше ветвистое дерево коммитов.

```
git log --oneline --graph
```

У вас должно получиться что-то вроде этого:

```
*   61b5884 (HEAD -> human) Merge yoda to human
|\  
| * 8efe6b8 (yoda) Цитата о вопросах и ответах
* | cf721a5 Цитата об ответах и вопросах
|/  
* 32a3cd9 (master, feature/megre, feature/branches) Добавлена информация о merge
*   6ca257d (origin/master) Merge pull request #2 from username/feature/branches
|\  
| * 16e30f8 (origin/feature/branches) Добавлена информация о ветвлении
|/  
* add34c9 Добавлена информация о команде reset
* 2b6a10d Добавлена информация о команде log
* 689a900 Добавлена информация о команде commit
* 91d0e04 Добавлен README.md
```
