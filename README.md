# Nootropic

Прокси-сервер, который принимает запросы в формате Anthropic API и транслирует их в OpenAI-совместимые вызовы. Позволяет приложениям, использующим Anthropic SDK, работать через OpenAI, Groq, OpenRouter или любой OpenAI-совместимый эндпоинт.

## Возможности

- Drop-in замена Anthropic API для любого клиента
- Стриминг (SSE) с корректной трансляцией чанков
- Tool calling (Anthropic tool_use -> OpenAI function calling)
- Мультимодальность (base64-изображения)
- Маршрутизация по моделям: несколько провайдеров в одном конфиге
- Интерактивный CLI-редактор конфигурации

## Быстрый старт

### Локально

```bash
npm install
npm run config          # интерактивная настройка
npm run dev             # dev-сервер с hot reload
```

### Docker Compose

```bash
cp config.example.toml config.toml
# отредактируй config.toml — укажи ключи и модели
docker compose up -d
```

Подробнее — в разделе [Docker Compose](#docker-compose-1).

## Использование

Прокси полностью имитирует Anthropic API. Используй его как обычный Anthropic-эндпоинт:

```bash
curl -X POST http://localhost:3000/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: any-key" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "claude-3-sonnet-20240229",
    "max_tokens": 1024,
    "messages": [{"role": "user", "content": "Hello!"}]
  }'
```

### Node.js

```javascript
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic({
  baseURL: 'http://localhost:3000',
  apiKey: 'any-key',
});

const msg = await client.messages.create({
  model: 'claude-3-sonnet-20240229',
  max_tokens: 1024,
  messages: [{ role: 'user', content: 'Hello!' }],
});
```

### Python

```python
import anthropic

client = anthropic.Anthropic(base_url="http://localhost:3000", api_key="any-key")

msg = client.messages.create(
    model="claude-3-sonnet-20240229",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello!"}],
)
```

`x-api-key` может быть любым — реальные ключи провайдеров хранятся в конфиге.

## Эндпоинты

| Метод | Путь | Описание |
|-------|------|----------|
| `POST` | `/v1/messages` | Создание сообщения (Anthropic-формат) |
| `GET` | `/v1/models` | Список моделей |
| `GET` | `/v1/models/:id` | Информация о модели |
| `GET` | `/health` | Health check |
| `GET` | `/` | Информация о сервере |

## Конфигурация

Конфиг хранится в `~/.config/nootropic/config.toml` (XDG). Создать или отредактировать:

```bash
npm run config
```

### Структура конфига

```toml
[server]
port = 3000
host = "localhost"

[server.cors]
enabled = true
origins = ["*"]

[[models]]
display_name = "gpt-4o"
provider = "openai"

[models.config]
base_url = "https://api.openai.com"
api_key = "sk-..."
model_name = "gpt-4o"
max_tokens = 128000              # опционально

[model_routing]
default_model_display_name = "gpt-4o"
route_claude_models_to_default = true

[defaults]
max_tokens = 4096
temperature = 0.7
stream = false

[logging]
enabled = true
level = "info"
format = "text"
```

### Поля моделей

| Поле | Описание |
|------|----------|
| `display_name` | Идентификатор модели для Anthropic-клиентов |
| `provider` | Тип провайдера (openai, groq, openrouter, custom) |
| `config.base_url` | OpenAI-совместимый эндпоинт |
| `config.api_key` | API-ключ провайдера |
| `config.model_name` | Имя модели на стороне провайдера |
| `config.max_tokens` | Переопределение max_tokens (опционально) |

### Маршрутизация

- `default_model_display_name` — модель по умолчанию
- `route_claude_models_to_default` — перенаправлять запросы с неизвестными `claude-*` моделями на дефолтную

## Docker Compose

### Требования

- Docker и Docker Compose

### Запуск

1. Создай `config.toml` из примера:

```bash
cp config.example.toml config.toml
```

2. Отредактируй `config.toml`:
   - Укажи реальные API-ключи провайдеров в `models.config.api_key`
   - Настрой нужные модели
   - Установи `host = "0.0.0.0"` в секции `[server]` (обязательно для Docker)

3. Запусти:

```bash
docker compose up -d
```

4. Проверь:

```bash
# health check
curl http://localhost:3000/health

# список моделей
curl http://localhost:3000/v1/models

# тестовый запрос
curl -X POST http://localhost:3000/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: any-key" \
  -d '{
    "model": "claude-3-sonnet-20240229",
    "max_tokens": 100,
    "messages": [{"role": "user", "content": "Hi"}]
  }'
```

### Управление

```bash
docker compose up -d      # запуск в фоне
docker compose logs -f     # логи
docker compose restart     # перезапуск
docker compose down        # остановка
docker compose build       # пересборка образа
```

### Конфигурация в Docker

Файл `config.toml` монтируется в контейнер как read-only volume. При изменении конфига перезапусти контейнер:

```bash
docker compose restart
```

Важно: в секции `[server]` конфига укажи `host = "0.0.0.0"`, иначе сервер будет слушать только localhost внутри контейнера и не будет доступен снаружи.

### Смена порта

Измени маппинг в `docker-compose.yml`:

```yaml
ports:
  - "8080:3000"   # доступен на хосте по порту 8080
```

## Разработка

```bash
npm run dev          # dev-сервер с hot reload
npm run build        # сборка
npm start            # production
npm test             # тесты
npm run lint         # линтер
npm run typecheck    # проверка типов
npm run config       # редактор конфигурации
```

### Структура проекта

```
src/
├── index.ts                    # точка входа
├── routes/
│   ├── messages.ts             # /v1/messages
│   └── models.ts               # /v1/models
├── services/
│   ├── translation.ts          # Anthropic <-> OpenAI трансляция
│   ├── openai.ts               # OpenAI API клиент
│   └── streaming-tool-state.ts # стейт стриминга tool calls
├── middleware/
│   ├── error-handler.ts        # обработка ошибок
│   └── request-validation.ts   # валидация запросов
├── utils/
│   ├── config.ts               # менеджер конфигурации
│   └── logger.ts               # логгер
├── config-editor/              # CLI-редактор конфига
└── types/
    └── index.ts                # TypeScript типы
```

## Лицензия

MIT
