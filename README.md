# tg-bitrix24-leads

Заявки из Telegram-бота в CRM Битрикс24 без дублей. Интеграция собрана на n8n и использует REST API Битрикс24 через входящий вебхук.

| Workflow | Файл | Назначение |
|---|---|---|
| **TG → Bitrix lead** | `workflows/01-tg-to-bitrix-lead.json` | Бот принимает контакт, ищет дубль, создаёт лид или добавляет комментарий к найденному, уведомляет менеджера |
| **Error alerts** | `workflows/02-error-alerts.json` | Error Workflow: присылает алерт об ошибке в Telegram |
| **Bitrix lead stage → TG client** | `workflows/03-bitrix-lead-stage-to-tg.json` | Исходящий вебхук `ONCRMLEADUPDATE`: при смене стадии лида сообщает клиенту новый статус |

## Схема потока

```
Telegram Trigger (message)
  └─ Config  ── Data Table integration_config (вебхук, chat_id менеджера)
      └─ Prepare (Code): разбор апдейта, нормализация телефона → +7XXXXXXXXXX
          └─ Route (Switch)
              ├─ /start    → Send Welcome: приветствие + reply-клавиатура «📱 Оставить заявку» (request_contact)
              ├─ contact   → Find Duplicate: crm.duplicate.findbycomm (LEAD, PHONE)
              │                └─ Is Duplicate?
              │                    ├─ да  → crm.timeline.comment.add «Повторное обращение из Telegram <дата>»
              │                    │        → Result: Duplicate ─┐
              │                    └─ нет → crm.lead.add          │
              │                             → Result: New ────────┤
              │                                                   ├─ Reply to Client («Заявка №ID принята» / «уже есть»)
              │                                                   └─ Notify Manager (тип, имя, телефон, ID и ссылка на лид)
              └─ прочее    → Send Hint: подсказка нажать кнопку
```

Поля, с которыми создаётся лид: `TITLE` = «Заявка из Telegram: <имя>», `NAME`, `LAST_NAME` (если есть), `PHONE` (`VALUE_TYPE=WORK`), `SOURCE_ID=TELEGRAM`, `COMMENTS` = «Telegram chat_id: <id>, username: @<username>».

## Защита от дублей

1. **Единый формат телефона.** Номер из контакта Telegram приводится к виду `+7XXXXXXXXXX`: из `8XXXXXXXXXX`, `7XXXXXXXXXX` и десятизначного номера получается одно и то же значение. Иностранные номера сохраняются как `+<цифры>`.
2. **Поиск перед созданием.** Перед `crm.lead.add` вызывается `crm.duplicate.findbycomm` с `entity_type=LEAD`, `type=PHONE`. Этот метод ищет по нормализованному индексу коммуникаций самого Битрикса, поэтому находит и лиды, созданные вручную или из других каналов.
3. **Повторное обращение не создаёт лид.** К найденному лиду (если их несколько, берётся самый свежий по ID) добавляется комментарий в таймлайн, клиент получает ответ «заявка уже есть», менеджер получает уведомление «🔁 Повторное обращение».
4. **Принимается только свой контакт.** Пересланный чужой контакт (`contact.user_id ≠ from.id`) отклоняется, бот отвечает подсказкой.

Остаточный риск: если клиент нажмёт кнопку два раза почти одновременно, оба execution могут успеть выполнить поиск раньше, чем первый создаст лид. Для демо-бота с одной кнопкой это маловероятно. В продакшене это закрывается блокировкой по телефону, например строкой в Data Table или Redis `SETNX` перед `crm.lead.add`.

## Повторы и алерты

- **Retry On Fail** включён на всех HTTP-нодах к Битриксу: 3 попытки, пауза 2000 мс. Это закрывает сетевые сбои, 5xx и `QUERY_LIMIT_EXCEEDED`.
- **Error Workflow.** У workflow 1 и 3 в настройках указан «Error alerts». При любой необработанной ошибке он присылает в Telegram менеджеру имя workflow, ноду, текст ошибки и ссылку на execution.
- **Перед отправкой алерта текст ошибки маскируется:** путь `/rest/<id>/<token>` и токены ботов Telegram заменяются на `***`. Затем текст экранируется под `parse_mode=HTML`, чтобы символы `_` и `*` в сообщении не ломали отправку.

## Где хранятся секреты

| Секрет | Где хранится | Попадает в экспорт |
|---|---|---|
| Токен Telegram-бота | n8n Credential `Telegram API`, зашифрован ключом n8n | Нет, в JSON только ссылка на credential |
| URL входящего вебхука Битрикс24 | n8n Data Table `integration_config`, ключ `bitrix_webhook` | Нет, в JSON только ID таблицы |
| chat_id менеджера | `integration_config`, ключ `manager_chat_id` | Нет |
| `application_token` исходящего вебхука | `integration_config`, ключ `bitrix_outgoing_token` | Нет |

Почему Data Table, а не credential. У входящего вебхука Битрикс24 токен находится в **пути** URL (`/rest/<user>/<token>/method`). Стандартные credentials n8n (Header, Query, Custom Auth) умеют передавать секрет только в заголовках, query или body, но не в пути. Data Table даёт одно место хранения: вебхук меняется в одной строке, все workflow сразу подхватывают новое значение.

Рассмотренные альтернативы:
- `$env` — нужен доступ к серверу и `N8N_BLOCK_ENV_ACCESS_IN_NODE=false`, после чего выражениям становятся доступны *все* переменные окружения;
- `$vars` — работает только в платных планах n8n.

Ограничение: в Data Table значение хранится в БД n8n незашифрованным и видно пользователям проекта n8n. Для продакшена лучше OAuth-приложение Битрикс24 (локальное или тиражное) с токенами в зашифрованном credential, либо внешний vault.

Файлы в `workflows/` прошли очистку: ID credential, Data Table и error workflow заменены на плейсхолдеры `REPLACE_*`. Отсутствие секретов проверено grep'ом.

## Как развернуть

1. **Битрикс24.** Откройте *Разработчикам → Другое → Входящий вебхук* и создайте вебхук с правом `crm`. CRM должна работать в классическом режиме, с лидами. Добавьте источник `TELEGRAM`:
   ```
   crm.status.add  fields[ENTITY_ID]=SOURCE  fields[STATUS_ID]=TELEGRAM  fields[NAME]=Telegram
   ```
2. **n8n → Credentials.** Создайте `Telegram API` с токеном бота от @BotFather.
3. **n8n → Data Tables.**
   - `integration_config` с колонками `key` (string) и `value` (string) и строками:
     | key | value |
     |---|---|
     | `bitrix_webhook` | `https://<portal>.bitrix24.ru/rest/<user>/<token>/` |
     | `manager_chat_id` | chat_id менеджера в Telegram |
     | `bitrix_outgoing_token` | application_token исходящего вебхука (только для workflow 3) |
   - `lead_stage` с колонками `lead_id` (number) и `status_id` (string), только для workflow 3.
4. **Импорт.** Импортируйте `workflows/*.json` и замените плейсхолдеры:
   - `REPLACE_TELEGRAM_CREDENTIAL_ID`: выберите credential в Telegram-нодах;
   - `REPLACE_CONFIG_TABLE_ID`, `REPLACE_LEAD_STAGE_TABLE_ID`: выберите таблицы в нодах Data Table;
   - `REPLACE_ERROR_WORKFLOW_ID`: *Settings → Error Workflow → Error alerts*.
5. **Активация.** Активируйте workflow. Telegram Trigger сам зарегистрирует webhook бота.

### Workflow 3: исходящий вебхук Битрикс24

1. В портале откройте *Разработчикам → Другое → Исходящий вебхук* и укажите:
   - **URL обработчика:** `https://<n8n-host>/webhook/bitrix24-lead-update`
   - **Событие:** `ONCRMLEADUPDATE` (Обновление лида)
2. Сохраните. Скопируйте **токен приложения** (`application_token`) и запишите его в `integration_config` под ключом `bitrix_outgoing_token`.

Как работает workflow 3. Битрикс передаёт в событии только ID лида. Workflow:
1. проверяет `application_token` и молча отбрасывает запросы без верного токена;
2. читает лид через `crm.lead.get`;
3. сравнивает `STATUS_ID` с последней известной стадией из `lead_stage`, по умолчанию `NEW`;
4. сохраняет новую стадию;
5. если стадия изменилась, достаёт `chat_id` клиента из `COMMENTS` лида и отправляет клиенту в Telegram название стадии из `crm.status.entity.items`.
