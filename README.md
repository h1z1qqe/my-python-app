# Pipeline CI на Go в GitHub Actions

[![CI for Go App](https://github.com/h1z1qqe/my-go-app/actions/workflows/ci.yml/badge.svg)](https://github.com/h1z1qqe/my-go-app/actions/workflows/ci.yml)

---

## Цель работы

**Цель** — учебный пример: простой проект, который можно склонировать, настроить и убедиться, что приложение работает в контейнере на **Go**, а **GitHub Actions** выполняет CI-пайплайн.

В ходе работы я научился:

- Настраивать **CI** для **Go**-проектов
- Контейнеризировать приложения с помощью **Docker**
- Собирать **Docker**-образ
- Сохранять артефакты для локального использования

**Go** (Golang) — это компилируемый, многопоточный язык программирования от **Google** с открытым исходным кодом, созданный для разработки высокопроизводительных веб-сервисов, микросервисов и облачных инфраструктур. Он сочетает синтаксис, похожий на **C**, с простотой и высокой скоростью выполнения.

---

## Выполнение работы

### 1. Создание репозитория и структуры проекта

На **GitHub** был создан новый публичный репозиторий `my-go-app` с файлом `README.md`.

Для работы над проектом я использовал **GitHub Codespaces** — облачную среду разработки, которая позволяет писать код и выполнять команды прямо в браузере, без необходимости устанавливать что-либо на локальный компьютер. Все дальнейшие действия выполнялись в терминале Codespaces.

Была создана следующая структура проекта:

```
my-go-app/
├── .github/
│   └── workflows/
│       └── ci.yml
├── main.go
├── sum.go
├── sum_test.go
├── Dockerfile
└── README.md
```

Структуру проекта можно создать одной **bash**-командой:

```shell
mkdir -p .github/workflows && \
touch .github/workflows/ci.yml \
      main.go sum.go sum_test.go \
      Dockerfile README.md
```

### 2. Инициализация Go-модуля

Так как **Go** в моей ОС не был установлен, я использовал **Docker** для инициализации модуля:

**Любой Unix (включая Codespaces):**

```shell
docker run --rm -v "$(pwd):/app" -w /app golang:1.22-alpine go mod init my-go-app
```

Затем для удовлетворения зависимостей:

```shell
docker run --rm -v "$(pwd):/app" -w /app golang:1.22-alpine go mod tidy
```

### 3. Файл `sum.go` (простая функция)

```go
package main

func Sum(a, b int) int {
    return a + b
}
```

### 4. Файл `main.go`

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello from Go app!")
    fmt.Println("2 + 3 =", Sum(2, 3))
}
```

### 5. Файл `sum_test.go` (тесты)

```go
package main

import "testing"

func TestSum(t *testing.T) {
    tests := []struct {
        a, b, expected int
    }{
        {2, 3, 5},
        {-1, 1, 0},
        {0, 0, 0},
    }
    for _, tt := range tests {
        result := Sum(tt.a, tt.b)
        if result != tt.expected {
            t.Errorf("Sum(%d, %d) = %d; want %d", tt.a, tt.b, result, tt.expected)
        }
    }
}
```

### 6. Файл `Dockerfile`

```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod .
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o my-app .

FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /app/my-app .
CMD ["./my-app"]
```

**Разбор Dockerfile:**

- **Стадия `builder`** — использует образ `golang:1.22-alpine` для сборки приложения.
- `COPY go.mod .` и `RUN go mod download` — копируют файл зависимостей и скачивают их.
- `RUN CGO_ENABLED=0 GOOS=linux go build -o my-app .` — компилирует бинарный файл с отключённым CGO для статической сборки.
- **Стадия финального образа** — использует лёгкий образ `alpine:latest`, устанавливает сертификаты и копирует готовый бинарник.
- `CMD ["./my-app"]` — запускает приложение при старте контейнера.

### 7. Файл GitHub Actions CI `.github/workflows/ci.yml`

```yaml
name: CI for Go App

on:
  push:
    branches: [ main, master ]
  pull_request:
    branches: [ main, master ]

jobs:
  test:
    name: Lint & Test
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.22'
          cache: true   # кэширует go mod

      - name: Download dependencies
        run: go mod download

      - name: Run golangci-lint
        uses: golangci/golangci-lint-action@v6
        with:
          version: latest
          args: --timeout=5m

      - name: Run tests with coverage
        run: go test -v -coverprofile=coverage.out ./...

      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage.out

  docker-build:
    name: Build Docker Image (no push)
    runs-on: ubuntu-latest
    needs: test
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4
      - name: Build Docker image
        run: docker build -t my-go-app:test .
```

**Разбор workflow:**

- **Job `test`** — выполняет линтинг и тестирование:
  - `actions/checkout@v4` — скачивает код репозитория.
  - `actions/setup-go@v5` с `cache: true` — устанавливает Go 1.22 и включает встроенное кэширование зависимостей.
  - `go mod download` — скачивает зависимости.
  - `golangci-lint-action@v6` — запускает линтер с таймаутом 5 минут.
  - `go test -v -coverprofile=coverage.out ./...` — запускает тесты с формированием отчёта о покрытии.
  - `actions/upload-artifact@v4` — сохраняет файл `coverage.out` как артефакт.

- **Job `docker-build`** — собирает **Docker**-образ без публикации:
  - Запускается только после успешного завершения job `test` (`needs: test`).
  - Выполняется только при пуше в ветку `main`.
  - `docker build -t my-go-app:test .` — собирает образ для проверки.

### 8. Запуск локальных тестов через Docker

Чтобы убедиться, что код приложения работает, я запустил тесты локально через **Docker**:

```shell
docker run --rm -v "$(pwd):/app" -w /app golang:1.22-alpine go test ./...
```

Если всё нормально, тесты показывают что-то вроде:

```shell
ok      my-go-app       0.002s
```

### 9. Сборка бинарного файла локально

```shell
docker run --rm -v "$(pwd):/app" -w /app golang:1.22-alpine go build -o my-app .
```

Бинарник программы на **Go** появляется в текущей папке. Запустить его можно через **Docker**:

```shell
docker run --rm -v "$(pwd):/app" -w /app alpine ./my-app
```

Если всё нормально, вывод будет таким:

```shell
Hello from Go app!
2 + 3 = 5
```

![Hello from my Go app!](/content/DevOps/CI_CD/img/6_workflow.png)

### 10. Проверка сборки онлайн

Я закоммитил и запушил файлы в ветку `main`:

```shell
git add .
git commit -m "Добавлен CI workflow для Go-приложения"
git push origin main
```

После пуша я перешёл на вкладку **Actions** в репозитории на **GitHub**. Там отобразился запущенный workflow. Через несколько минут загорелась **зелёная галочка** — все шаги прошли успешно.

![Скриншот успешного запуска workflow](/content/DevOps/CI_CD/img/5_workflow.png)

### 11. Проверка сборки Docker-образа локально

Находясь в папке `my-go-app`, я выполнил сборку проекта в **Docker**-образ:

```shell
docker build -t my-go-app:latest .
```

Создание и запуск контейнера:

```shell
docker run --rm my-go-app:latest
```

Вывод:

```shell
Hello from Go app!
2 + 3 = 5
```

![Hello from my Go app!](/content/DevOps/CI_CD/img/7_workflow.png)

Опционально можно зайти в интерактивный режим контейнера для отладки:

```shell
docker run -it --rm my-go-app:latest /bin/sh
```

Получить информацию об ОС в контейнере:

```shell
cat /etc/os-release
```

![Информация об ОС в контейнере](/content/DevOps/CI_CD/img/8_workflow.png)

Выйти из контейнера:

```shell
exit
```

---

## Использованные инструменты

- **GitHub** — платформа для хостинга репозиториев.
- **GitHub Codespaces** — облачная среда разработки.
- **GitHub Actions** — система CI/CD для автоматизации сборки, тестирования и развёртывания.
- **Go (Golang)** — компилируемый язык программирования для высокопроизводительных сервисов.
- **Docker** — платформа контейнеризации приложений.
- **YAML** — язык разметки для конфигурационных файлов.
- **Markdown** — язык разметки для оформления документации.

---

## Заключение

В ходе работы я успешно настроил CI-пайплайн для **Go**-приложения с помощью **GitHub Actions**. Workflow автоматически запускается при пуше и создании pull request, выполняет линтинг, тестирование с отчётом о покрытии и сборку **Docker**-образа. Все шаги прошли успешно, что подтверждается зелёной галочкой на вкладке **Actions**. Я также освоил контейнеризацию **Go**-приложений с помощью **Docker** и научился сохранять артефакты для локального использования.
