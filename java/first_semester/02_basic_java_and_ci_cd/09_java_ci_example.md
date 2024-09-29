# Пример CI на Java

В прошлом модуле мы рассмотрели с вами основы Git.
Давайте сохраним ту работу, что мы уже сделали, в репозитории и настроим там CI-процесс для сборки нашего проекта.

Откройте репозиторий, который вы создали в прошлом модуле.
Давайте немного отредактируем `.gitignore` (мы обсуждали смысл этого файла в прошлом модуле). Его контент вы можете посмотреть ниже.
Здесь мы игнорируем содержимое папки `target`, где находятся бинарники (результат компиляции программы), и оставляем только исходный код.

> Если вы читаете другую версию курса, то этот модуль для вас – первый. Тогда создайте новый репозиторий на GitHub.

<details>
<summary>Содержимое .gitignore</summary>

```
*#
*.iml
*.ipr
*.iws
*.jar
*.sw?
*~
.#*
.*.md.html
.DS_Store
.attach_pid*
.classpath
.factorypath
.gradle
.idea
.metadata
.project
.recommenders
.settings
.springBeans
.vscode
/code
MANIFEST.MF
_site/
activemq-data
bin
build
!/**/src/**/bin
!/**/src/**/build
build.log
dependency-reduced-pom.xml
dump.rdb
interpolated*.xml
lib/
manifest.yml
out
overridedb.*
target
.flattened-pom.xml
secrets.yml
.gradletasknamecache
.sts4-cache
```
</details>

Также стоит добавить еще файл `.gitattributes`. Он нужен для того, чтобы автоматически преобразовывать переносы строк в `LF`,
потому что на Windows и Linux/Mac они отличаются. Пример содержимого:

```
* text=auto
```

Теперь сделайте коммит и запушьте то, что вы сделали на Java в рамках этого модуля. Что же, осталось лишь настроить CI-процесс.
В прошлом модуле мы рассматривали пример с созданием простого файла `.github/workflows/build.yml`. Добавьте такой же.
Правда, содержимое будет другим:

```yaml
name: Java CI with Maven

on:
  push:
    branches: [ "master" ]
  pull_request:
    branches: [ "master" ]

jobs:
  build:

    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: maven
      - name: Build with Maven
        run: mvn package
```

Здесь мы выполняем следующие шаги:

1. Устанавливаем Java 17.
2. Запускаем команду `mvn package`, которая и собирает проект.

Если вы всё сделали правильно, то увидите вот такой желтый значок рядом с количеством коммитов:

![ci building example](img/ci-building-example.png)

Желтый цвет означает, что сборка идет. Вы можете нажать на иконку, чтобы посмотреть логи.
Когда значок сменится на зеленую галочку, это будет значить, что билд прошел успешно.

Если у вас возникли затруднения, вы можете посмотреть на пример [этого тестового проекта](https://github.com/SimonHarmonicMinor/java-maven-ci-example).