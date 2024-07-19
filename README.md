# go-autotests

Автотесты для курса «Продвинутый Go-разработчик для сетевых инженеров».

## Локальный запуск

1. Скомпилируйте ваши сервер и агент в папках `cmd/server` и `cmd/agent` командами `go build -o server *.go` и `go build -o agent *.go` соответственно.
2. Скачайте [бинарный файл с автотестами](https://github.com/mipt-golang-course/go-autotests/releases/latest) для вашей ОС — например, `metricstest-darwin-arm64` для MacOS на процессоре Apple Silicon.
3. Разместите бинарный файл так, чтобы он был доступен для запуска из командной строки, — пропишите путь в переменную `$PATH`.
4. Ознакомьтесь с параметрами запуска автотестов в файле `.github/workflows/metricstest.yml` вашего репозитория. Автотесты для разных инкрементов требуют различных аргументов для запуска.

Пример запуска теста для первого инкремента:

```shell
metricstest -test.v -test.run=^TestIteration1$ -agent-binary-path=cmd/agent/agent
```

## Запуск на Mac с процессором Apple Silicon

Если у вас возникают трудности с локальным запуском автотестов на компьютере Mac на базе процессора Apple Silicon (M1 и старше), убедитесь, что:

- Вы запускаете бинарный файл с суффиксом `darwin-arm64`.
- У файла выставлен флаг на исполнение командой `chmod +x <filename>`.
- Вы разрешили выполнение неподписанного бинарного файл в разделе «Безопасность» настроек системы.

## Автоматический запуск

Для проверки того, насколько код инкремента удовлетворяет функциональным требованиям задания, мы используем автотесты. Они запускаются внутри CI/CD-инструмента под названием **GitHub Actions**.

GitHub Actions (далее GA) позволяет с помощью собственного синтаксиса описывать некоторый набор операций (`workflow`), которые должны применяться к коду автоматически при изменениях или при нажатии кнопки. На практике с помощью CI/CD часто автоматизируют синтаксические анализаторы, автотесты, генерацию документации, выкатку на различные окружения и так далее. При этом все эти действия выполняются не в вакууме: в нашем случае GitHub предоставляет свои машины (runner'ы), где код и будет выполняться в соответствии с заготовленными конфигурациями.

Если вкратце, GA разбивает каждый `workflow` на несколько `job`, которые могут работать последовательно или параллельно. Джобы в свою очередь состоят из одного или нескольких `steps`, где можно указывать конкретные действия, которые должны быть выполнены. Также вы можете указать условия выполнения `workflow` (например, на каждый коммит в ветке `main`), что позволяет гибко настраивать ваши самые смелые CI/CD-фантазии.

---

<details>
  <summary>Пример workflow-конфига, который мы используем для автоматического применения go vet</summary>

```yaml
name: go vet test # название workflow
# если вы хотите настроить автоматическое применение workflow в зависимости от условий,
# то используйте ключевое слово on
on:
  pull_request: # здесь говорите, что workflow должен запускаться для любого события внутри PR (пуш, тег и другие)
  push:
    branches: # здесь говорите, что хотите применять workflow и для пушей в main-ветку
      - main
# каждый workflow представляет собой набор джоб,
# которые выполняются последовательно или параллельно,
# для каждой джобы можно задать докер-образ, в котором будут выполняться шаги (steps),
# и ОС, в которой будет запущен контейнер
jobs:
  statictest: # описываете джобу statictest
    runs-on: ubuntu-latest # говорите, что джоба должна выполняться на машине с убунтой (предоставляется гитхабом)
    container: golang:1.22 # запускаете в ней контейнер докер-образа golang:1.22
    steps: # последовательно выполняемые шаги
      - name: Checkout code
        # GitHub Actions позволяет самим описывать команды линукса внутри шага
        # или использовать заготовленные шаги, как здесь
        uses: actions/checkout@v2
      - name: Download statictest binary
        # или здесь
        uses: robinraju/release-downloader@v1.8
        with:
        # для заготовленных шагов иногда требуется указать параметры, как здесь
          repository: mipt-golang-course/go-autotests
          latest: true
          fileName: statictest
          out-file-path: .tools
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Setup autotest binary
        # тут описываете произвольный набор команд
        run: |
          chmod -R +x $GITHUB_WORKSPACE/.tools/statictest
          mv $GITHUB_WORKSPACE/.tools/statictest /usr/local/bin/statictest
      - name: Run statictest
        run: |
          go vet -vettool=$(which statictest) ./...
```
</details>

---

Работает это следующим образом:

1. Когда вы подтягиваете шаблон проекта, вместе с ним подтягивается директория `.github/workflows`. В ней находятся два `workflow`-файла: `statictest.yaml` и `metricstest.yaml`.

2. В следующий раз, когда вы выполните изменения в репозитории (например, запушите коммит), которые попадают под условия, описанные в `on` конфигов `workflow` в папке `.github`, GA выполнит все джобы в каждом файле (по умолчанию — параллельно).

3. Вы можете следить за ходом выполнения джоб в интерфейсе Pull Request или перейти во вкладку `Actions`.

4. Слева во вкладке `Actions` есть список `workflow`. Например, если вы хотите посмотреть на ход выполнения автотестов, переходите в `autotests`.

5. Перейдя в интересующий вас `workflow`, вы увидите страницу с историей выполнения только этого `workflow`. Чаще всего вас интересует последний, так как он, вероятно, был выполнен на самой актуальной версии кода. Кстати, не удивляйтесь тому, что `workflow` помечен красной иконкой, — она сменится на приятную глазу зелёную, только когда вы выполните все инкременты курса.

6. Если вы зайдёте в последний выполненный автотест `workflow`, то увидите несколько джоб.

7. Возможно, с первого раза ваш код не пройдёт автотесты :)

8. После того как у вас завалится какой-то из шагов джобы, джоба завершится, а все контейнеры и их содержимое удалятся.
