# Сокращатель URL

## Обзор

Этот проект представляет собой простой сервис для сокращения URL, написанный на Go. Он позволяет пользователям сокращать URL и перенаправляет их на исходные URL при доступе через сокращенные пути. Сервис поддерживает сопоставление путей с URL, используя как жестко закодированную карту, так и конфигурацию в YAML.

## Возможности

- **Сокращение URL на основе карты**: Перенаправляет на основе предопределенной карты путей к URL.
- **Сокращение URL на основе YAML**: Перенаправляет на основе путей и URL, указанных в YAML файле.
- **Обработка резервных вариантов**: Обеспечивает механизм резервного решения для обработки путей, не определенных в карте или YAML файле.

## Начало работы

### Установка
0. Установите Go

1. Клонируйте репозиторий:
    ```sh
    git clone https://github.com/nemopss/urlshort.git
    cd urlshort
    ```

2. Установите зависимости:
    ```sh
    go mod tidy
    ```

### Запуск сервера

1. Чтобы запустить сервер, выполните:
    ```sh
    go run main.go
    ```

2. Сервер будет запущен на порту `8080`. Вы можете получить к нему доступ по адресу `http://localhost:8080`.

## Использование

### Сокращение URL на основе карты

`MapHandler` сопоставляет предопределенные пути с URL. Эти сопоставления определены в файле `main.go`:

```go
pathsToUrls := map[string]string{
    "/urlshort-godoc": "https://godoc.org/github.com/gophercises/urlshort",
    "/yaml-godoc":     "https://godoc.org/gopkg.in/yaml.v2",
}
```

### Сокращение URL на основе YAML

`YAMLHandler` читает пути и URL из конфигурации YAML:

```yaml
yaml := `
- path: /urlshort
  url: https://github.com/gophercises/urlshort
- path: /urlshort-final
  url: https://github.com/gophercises/urlshort/tree/solution
`
```

### Резервный обработчик

Любые пути, не определенные в карте или YAML, будут обрабатываться резервным обработчиком, который отвечает "Hello, world!".

## Структура кода

- `main.go`: Точка входа приложения, настройка сервера и обработчиков.
- `handler.go`: Содержит реализацию `MapHandler` и `YAMLHandler`.

### main.go

Настраивает сервер и обработчики:

```go
package main

import (
    "fmt"
    "net/http"
    "github.com/nemopss/urlshort"
)

func main() {
    mux := defaultMux()

    pathsToUrls := map[string]string{
        "/urlshort-godoc": "https://godoc.org/github.com/gophercises/urlshort",
        "/yaml-godoc":     "https://godoc.org/gopkg.in/yaml.v2",
    }
    mapHandler := urlshort.MapHandler(pathsToUrls, mux)

    yaml := `
    - path: /urlshort
      url: https://github.com/gophercises/urlshort
    - path: /urlshort-final
      url: https://github.com/gophercises/urlshort/tree/solution
    `
    yamlHandler, err := urlshort.YAMLHandler([]byte(yaml), mapHandler)
    if err != nil {
        panic(err)
    }
    fmt.Println("Starting the server on :8080")
    http.ListenAndServe(":8080", yamlHandler)
}

func defaultMux() *http.ServeMux {
    mux := http.NewServeMux()
    mux.HandleFunc("/", hello)
    return mux
}

func hello(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "Hello, world!")
}
```

### handler.go

Содержит обработчики для конфигураций на основе карты и YAML:

```go
package urlshort

import (
    "net/http"
    "gopkg.in/yaml.v2"
)

func MapHandler(pathsToUrls map[string]string, fallback http.Handler) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        path := r.URL.Path
        if dest, ok := pathsToUrls[path]; ok {
            http.Redirect(w, r, dest, http.StatusFound)
            return
        }
        fallback.ServeHTTP(w, r)
    }
}

func YAMLHandler(yml []byte, fallback http.Handler) (http.HandlerFunc, error) {
    pathUrls, err := parseYaml(yml)
    if err != nil {
        return nil, err
    }
    pathsToUrls := buildMap(pathUrls)
    return MapHandler(pathsToUrls, fallback), nil
}

func buildMap(pathUrls []pathUrl) map[string]string {
    pathsToUrls := make(map[string]string)
    for _, pu := range pathUrls {
        pathsToUrls[pu.Path] = pu.URL
    }
    return pathsToUrls
}

func parseYaml(data []byte) ([]pathUrl, error) {
    var pathUrls []pathUrl
    err := yaml.Unmarshal(data, &pathUrls)
    if err != nil {
        return nil, err
    }
    return pathUrls, nil
}

type pathUrl struct {
    Path string `yaml:"path"`
    URL  string `yaml:"url"`
}
```
