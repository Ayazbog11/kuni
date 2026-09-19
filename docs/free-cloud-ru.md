# Облачный профиль без локальной LLM

Этот профиль рассчитан на небольшой VPS без GPU:

- Kuni остаётся Telegram-клиентом, памятью/RAG и системой инструментов;
- FreeDeepseekAPI предоставляет основную reasoning-модель через OpenAI-compatible API;
- Google Gemini API создаёт embeddings для памяти, поэтому Ollama не нужен;
- прокси FreeDeepseekAPI слушает только `127.0.0.1:9655` и не публикуется в Интернет.

Профиль подготовлен для снимка `alex2772/kuni` commit `3bbf87d6fc37091758e00b8f1834ff3b2ae6c1f0`. FreeDeepseekAPI закреплён на commit `31386b18cd485e5c6677bc0f46ac0ae3d92fbfd5`.

## Важные ограничения

FreeDeepseekAPI — не официальный бесплатный API-ключ. Он использует браузерную сессию вашего аккаунта на `chat.deepseek.com`. Внутренний Web API может измениться, сессия может истечь, а доступность и допустимость такого способа зависят от правил DeepSeek. Для критичного production лучше официальный API.

Файл `deepseek-auth.json` фактически даёт доступ к сессии DeepSeek. Не отправляйте его в чат, не публикуйте и не добавляйте в Git.

## 1. Авторизация DeepSeek на компьютере

Скачайте FreeDeepseekAPI и выполните в его каталоге:

```powershell
npm run auth
```

Откроется отдельный профиль Chrome. Войдите в свой аккаунт DeepSeek, отправьте короткое сообщение и вернитесь в терминал. В каталоге появится `deepseek-auth.json`.

## 2. Бесплатный API-ключ для embeddings

Создайте ключ Gemini API в Google AI Studio. Gemini имеет OpenAI-compatible endpoint `/v1/embeddings`; новые аккаунты начинают с Free Tier, но доступность и квота зависят от проекта и могут изменяться.

В `deploy/free-cloud/config.toml.example` замените:

- `REPLACE_GEMINI_API_KEY`;
- обязательные поля Telegram и владельца.

Модель `gemini-embedding-001` поддерживает многоязычный текст. После смены embedding-модели существующие embeddings нужно построить заново, потому что пространства разных моделей несовместимы.

Важно: на бесплатном тарифе Google может использовать отправленный контент для улучшения продуктов. Kuni отправляет провайдеру записи памяти и контекст разговоров, поэтому не используйте этот профиль для конфиденциальных переписок.

## 3. Секрет прокси

Создайте длинный случайный ключ. Например, в PowerShell:

```powershell
$proxyKeyBytes = New-Object byte[] 32
[Security.Cryptography.RandomNumberGenerator]::Fill($proxyKeyBytes)
$proxyKey = [Convert]::ToHexString($proxyKeyBytes)
$proxyKey
```

Одинаковое значение должно находиться:

1. в `deploy/free-cloud/secrets/proxy-api-key` на VPS;
2. в поле `general.llm.endpoint.bearerKey` файла `config.toml`.

## 4. Файлы на VPS

Структура должна выглядеть так:

```text
kuni/
├── deploy/free-cloud/compose.yaml
├── deploy/free-cloud/secrets/deepseek-auth.json
├── deploy/free-cloud/secrets/proxy-api-key
└── config.toml
```

Выставьте строгие права:

```bash
chmod 600 deploy/free-cloud/secrets/deepseek-auth.json
chmod 600 deploy/free-cloud/secrets/proxy-api-key
chmod 600 config.toml
```

Запустите прокси:

```bash
docker compose -f deploy/free-cloud/compose.yaml up -d --build
curl --fail http://127.0.0.1:9655/health
```

После этого запустите Kuni из каталога, где лежит `config.toml`.

## Резервный бесплатный LLM

Если Web API DeepSeek временно сломается, можно вручную переключить `general.llm` на OpenAI-compatible модель Gemini, используя тот же API key, что и для embeddings. Например:

```toml
llm = { endpoint = { baseUrl = "https://generativelanguage.googleapis.com/v1beta/openai/", bearerKey = "GEMINI_API_KEY" }, model = "gemini-3.8-flash" }
```

Это ручной fallback: Kuni подхватит изменение `config.toml` без сохранения ключей в репозитории. Перед использованием проверьте актуальную бесплатную квоту и доступность модели в Google AI Studio.
