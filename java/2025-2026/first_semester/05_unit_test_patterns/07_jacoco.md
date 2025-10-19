# Jacoco

Тесты полезны, когда они присутствуют. Если их мало, то и толку от них не очень много. В Java есть
инструменты, которые проверяют процент покрытия кода тестами и завершают билд с ошибкой, если этот процент
ниже некоторого заданного порогового значения.

Давайте добавим Jacoco в наш проект. Для это в `pom.xml` нужно будет добавить новый блок `plugin`:

```xml

<plugin>
  <groupId>org.jacoco</groupId>
  <artifactId>jacoco-maven-plugin</artifactId>
  <version>0.8.7</version>
  <executions>
    <execution>
      <goals>
        <goal>prepare-agent</goal>
      </goals>
    </execution>
    <execution>
      <id>report</id>
      <phase>prepare-package</phase>
      <goals>
        <goal>report</goal>
      </goals>
    </execution>
    <execution>
      <id>jacoco-check</id>
      <goals>
        <goal>check</goal>
      </goals>
      <configuration>
        <rules>
          <rule>
            <element>PACKAGE</element>
            <limits>
              <limit>
                <counter>LINE</counter>
                <value>COVEREDRATIO</value>
                <minimum>0.50</minimum>
              </limit>
            </limits>
          </rule>
        </rules>
      </configuration>
    </execution>
    <execution>
      <id>post-unit-test</id>
      <phase>test</phase>
      <goals>
        <goal>report</goal>
      </goals>
      <configuration>
        <dataFile>target/jacoco.exec</dataFile>
        <outputDirectory>target/jacoco-ut</outputDirectory>
      </configuration>
    </execution>
  </executions>
</plugin>
```

Здесь процент покрытия настраивается в блоке `<minimum>`, и он равен 50 процентам. Чтобы запустить сборку проекта
без проверки покрытия, запустите `test` или `package`. При этом в директории `target/jacoco-ut` сгенерируется отчет
о покрытии. Вы можете открыть в браузере `index.html`, чтобы посмотреть его.

Теперь выполните команду `verify`. Она идет следом за `package` и включает в себя выполнение тестов и сборку артефактов.
Скорее всего, команда завершится ошибкой, а в логах вы найдете примерно такие сообщения:

```
Rule violated for package org.example.action: lines covered ratio is 0.00, but expected minimum is 0.50
Rule violated for package org.example.exception: lines covered ratio is 0.33, but expected minimum is 0.50
```

Добавив всего несколько строчек в xml-файл, мы смогли автоматизировать проверку процента покрытия кода тестами.
Теперь давайте сделаем так, чтобы проверка выполнялась на CI. Откройте `.github/workflows/build.yml`,
который мы писали в прошлом модуле, и замените команду `mvn package` на `mvn verify`.
Сделайте Pull Request и проверьте, что при недостаточном количестве тестов билд падает автоматически и не дает
влить Pull Request в master-ветку.

> Если у вас возникли трудности с настройкой, посмотрите готовый пример [здесь](https://github.com/SimonHarmonicMinor/java-unit-tests-example).