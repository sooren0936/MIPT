# Полезные команды

- `git commit --amend <commit message>` — изменить сообщение последнего коммита.
- `git stash` — спрятать текущие изменения и откатить их. Удобно, если не хочется удалять изменения, но нужно откатить их на время ради переключения в другую ветку.
- `git pull` — получить изменения с сервера.
- `git log --graph --oneline --decorate` — отразить историю коммитов в виде графа.
- Фильтрация истории коммитов:
    - По дате:
        - `git log --after="2020-4-1"`
        - `git log --after="yesterday"`
        - `git log --after="1 week ago"`
        - `git log --after="2020-4-1" --before="2014-4-6"`
    - По автору:
        - `git log --author="Alex"` — найти коммиты Алекса.
        - `git log --author="Alex\|Ivan"` — найти коммиты Алекса и Ивана.
    - По другим параметрам:
        - `git log --grep="ISSUE-777:"` — по паттерну сообщения.
        - `git log -- [foo.kts](http://foo.kts/) [bar.kts](http://bar.kts/)` — по файлам, которые были изменены.
        - `git log -S "git"` — поиск по содержимому файлов. Например, когда в файлы была добавлена строчка `git`.
    - `git log --no-merges` — убрать из выборки мёрж-коммиты.
    - `git log --merges` — выбрать только мёрж-коммиты.

# Ссылки

- [Введение в основы Git](https://git-scm.com/book/ru/v2/Введение-Основы-Git)
- [Создание алиасов команд](https://git-scm.com/book/ru/v2/Основы-Git-Псевдонимы-в-Git)
- [Как просматривать историю изменений](https://www.atlassian.com/git/tutorials/git-log) на английском
- [git rebase для начинающих](https://habr.com/ru/post/337302/)
