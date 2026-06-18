# Блок 4: Тестирование API

Чек-лист для проверки корректного поведения контроллера `/api/v2/orders` с опорой на индустриальные стандарты (Google REST API Guide, Microsoft REST API Guidelines).

## Структура файлов

- `checklist.md` — чек-лист: статус-коды для успешных операций, ошибок валидации, отсутствия прав доступа, несуществующего ID, падения БД

---

# Тестовый стенд

Используя ИИ сгенерирован тестовый стенд БД + API

[https://github.com/elenafyodorova/api-test-sandbox/tree/main](https://github.com/elenafyodorova/api-test-sandbox/tree/main)


## Запуск проекта

```bash
docker-compose up --build
```

Swagger / OpenAPI документация: `http://localhost:8000/docs`

## Postman

### Коллекция запросов

`assets/Smart-Logistics-Orders-API.postman_collection.json`

### Окружение

`assets/Smart-Logistics-Local.postman_environment.json`


![Тестовый стенд API](./assets/test-stand.png)
