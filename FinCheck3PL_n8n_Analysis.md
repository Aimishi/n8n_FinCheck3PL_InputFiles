# [FinCheck3PL] - Анализ адаптации ТЗ для реализации на n8n

## 1. Резюме

Данное техническое задание было изначально разработано для платформы **RPA Primo**, но в связи с изменением архитектурного решения требуется реализовать процесс на платформе **n8n** (low-code платформа для автоматизации рабочих процессов на основе workflow).

### Ключевые изменения при переходе с Primo на n8n:

| Аспект | Primo RPA | n8n |
|--------|-----------|-----|
| **Архитектура** | Роботы (Robot 1, Robot 2) с очередями | Единый workflow с под-workflow |
| **Триггеры** | Очереди Primo Orchestrator | Webhook, Schedule, Manual trigger |
| **Хранение данных** | RPA БД (внешняя) | PostgreSQL/MySQL node + временное хранение |
| **Интеграция с MM** | Специфичные коннекторы Primo | HTTP Request node (Mattermost API) |
| **Работа с Excel** | Библиотеки Primo | Excel/Google Sheets nodes или Code node |
| **Логирование** | Встроенное в платформу | Custom logging через Code node + внешние системы |
| **Обработка ошибок** | Retry блоки Primo | Error Trigger + Retry logic в Code node |

---

## 2. Архитектурное решение для n8n

### 2.1 Общая схема процесса

```
┌─────────────────────────────────────────────────────────────────┐
│                    WORKFLOW: FinCheck3PL Main                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐   │
│  │   Webhook    │────▶│  Init Config │────▶│ Get Files    │   │
│  │  (Mattermost)│     │  (Settings)  │     │ from Storage │   │
│  └──────────────┘     └──────────────┘     └──────────────┘   │
│                                                  │              │
│                                                  ▼              │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐   │
│  │ Send Report  │◀────│ Validate Data│◀────│ Identify CD  │   │
│  │ (MM + Email) │     │  (Columns/   │     │ (Delivery    │   │
│  │              │     │   Sheets)    │     │  Service)    │   │
│  └──────────────┘     └──────────────┘     └──────────────┘   │
│         │                    │                                 │
│         │                    ▼                                 │
│         │           ┌──────────────┐                          │
│         │           │   Jupyter    │                          │
│         │           │ Integration  │                          │
│         │           │  (HTTP API)  │                          │
│         │           └──────────────┘                          │
│         │                    │                                 │
│         └────────────────────┴─────────────────────────────────┘
│
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Декомпозиция на workflow

Вместо двух роботов (Robot 1 и Robot 2) предлагается следующая структура:

#### **Workflow 1: FinCheck3PL_Trigger** (основной)
- **Триггер**: Webhook от Mattermost
- **Функции**:
  1. Инициализация (загрузка конфига)
  2. Получение файлов от пользователя
  3. Определение СД
  4. Проверка обязательных колонок/листов
  5. Валидация данных
  6. Сохранение данных во временное хранилище (БД)
  7. Запуск Workflow 2 (Jupyter integration)
  8. Формирование отчёта
  9. Отправка уведомлений (MM + Outlook)

#### **Workflow 2: FinCheck3PL_JupyterCheck** (дополнительный)
- **Триггер**: Вызов из Workflow 1
- **Функции**:
  1. Проверка заполненности строк
  2. Проверка статусов (для Яндекс Внутригород, Достависта)
  3. Проверка дат (для большинства СД)
  4. Проверка весов (для всех СД)
  5. Подготовка файлов для DWH/Jupyter
  6. Интеграция с Jupyter Notebook через API
  7. Возврат результатов в Workflow 1

---

## 3. Детальная карта соответствия шагов ТЗ → n8n Nodes

### 3.1 Robot 1 → Workflow 1

| Шаг ТЗ | Описание | n8n Node | Примечания |
|--------|----------|----------|------------|
| 1 | Старт процесса | **Webhook Node** | Триггер от Mattermost |
| 2 | Пройти инициализацию | **Code Node** + **Read/Write File** | Загрузка config.json с настройками |
| 3 | Получить файл от пользователя | **Webhook Response** + **Move File** | Файлы сохраняются в `/data/files/input/` |
| 4 | Определить СД | **Switch Node** | Выбор из dropdown (11 СД) |
| 5 | Проверить наличие обязательных колонок | **Code Node** (Python/JS) | Использование библиотеки `xlsx` для чтения Excel |
| 6 | Все обязательные колонки есть? | **IF Node** | Ветвление: Да → шаг 7, Нет → шаг 10 |
| 7 | Перенести данные в БД | **PostgreSQL/MySQL Node** | Insert данных из всех листов |
| 8 | Есть данные? | **IF Node** | Проверка количества записей |
| 9 | Установить элемент в очередь для робота 2 | **Execute Workflow Node** | Запуск Workflow 2 с передачей параметров |
| 10 | Сформировать и отправить сообщение | **HTTP Request Node** (MM) + **SMTP Node** (Outlook) | Отправка уведомлений по шаблонам |
| 11 | Завершение робота | **End Node** | Финализация с статусом Success/Error |

### 3.2 Robot 2 → Workflow 2

| Шаг ТЗ | Описание | n8n Node | Примечания |
|--------|----------|----------|------------|
| 1 | Старт процесса | **Workflow Trigger** | Вызов из Workflow 1 |
| 2 | Пройти инициализацию | **Code Node** | Загрузка конфига |
| 3 | Определить СД и время запуска | **Code Node** | Извлечение из входных данных |
| 4 | Проверка на заполнение строк | **Code Node** (SQL query) | Поиск пустых значений в обязательных колонках |
| 5 | Есть незаполненные строки? | **IF Node** | Ветвление |
| 6 | Дать признак незаполненным строкам | **PostgreSQL Node** (UPDATE) | Установка флага `is_filled_in_lines = true` |
| 7 | Проверка на статусы | **Code Node** + **Switch Node** | Логика индивидуальна для каждой СД |
| 8 | Нашлись заказы на проверке статусов? | **IF Node** | Ветвление |
| 9 | Перенести найденные заказы | **PostgreSQL Node** (UPDATE) | Установка флага `is_checking_for_statuses = true` |
| 10 | Проверка на даты | **Code Node** + **Switch Node** | Проверка дат текущего месяца |
| 11 | Нашлись заказы на проверке даты? | **IF Node** | Ветвление |
| 12 | Перенести найденные заказы | **PostgreSQL Node** (UPDATE) | Установка флага `is_checking_for_date = true` |
| 13 | Проверка на веса | **Code Node** + **Switch Node** | Проверка лимитов веса по СД |
| 14 | Нашлись заказы на проверке веса? | **IF Node** | Ветвление |
| 15 | Перенести найденные заказы | **PostgreSQL Node** (UPDATE) | Установка флага `is_checking_for_weights = true` |
| 16 | Выбрать строки для Jupiter | **PostgreSQL Node** (SELECT) | Фильтрация по СД и timestamp |
| 17 | Есть строки? | **IF Node** | Проверка наличия данных |
| 18 | Подготовить файлы для DWH | **Code Node** + **Write Binary File** | Переименование колонок, создание Excel |
| 19-23 | *Продолжение подготовки файлов* | **Code Node** | Маппинг колонок для каждой СД |
| 24 | Отправить файл в Jupyter | **HTTP Request Node** | POST запрос к Jupyter API |
| 25 | Получить результаты от Jupyter | **HTTP Request Node** | GET запрос с результатами |
| 26 | Обработать результаты | **Code Node** | Парсинг ответа Jupyter |
| 27 | Сформировать итоговый отчёт | **Code Node** + **Create Excel** | Генерация файла с ошибками |
| 28 | Отправить уведомления | **HTTP Request** (MM) + **SMTP** (Email) | Рассылка по маппингу |
| 29 | Завершение | **Return Node** | Возврат статуса в Workflow 1 |

---

## 4. Технические требования для реализации на n8n

### 4.1 Необходимые n8n Nodes

| Категория | Nodes | Версия |
|-----------|-------|--------|
| **Триггеры** | Webhook, Manual Trigger, Schedule Trigger | Latest |
| **Логика** | IF, Switch, Merge, Split In Batches | Latest |
| **Данные** | Code (JavaScript/Python), Edit Fields | Latest |
| **Файлы** | Read/Write Binary File, Move File | Latest |
| **Excel** | Google Sheets **или** Code Node + xlsx library | - |
| **Базы данных** | PostgreSQL, MySQL | Latest |
| **API** | HTTP Request | Latest |
| **Email** | SMTP, Outlook | Latest |
| **Очереди** | Execute Workflow, Wait Node | Latest |

### 4.2 Внешние зависимости

```json
{
  "dependencies": {
    "xlsx": "^0.18.0",
    "node-fetch": "^2.6.0",
    "pg": "^8.11.0",
    "nodemailer": "^6.9.0"
  }
}
```

### 4.3 Конфигурационный файл (config.json)

```json
{
  "deliveryServices": {
    "5post": {
      "requiredColumns": [
        "Дата и время получения заявки",
        "Физический(платный) вес, кг",
        "Стоимость за доставку отправления с объявленной ценностью, без НДС, руб.",
        "Стоимость за доставку отправления, без НДС, руб.",
        "Итоговая стоимость доставки отправления, без НДС, руб.",
        "Номер накладной",
        "Номер заказа клиента",
        "Тарифная зона",
        "Итоговая стоимость доставки, без НДС, руб."
      ],
      "weightLimit": 15,
      "dateColumn": "Дата и время получения заявки",
      "statusCheck": false
    },
    "yandexCity": {
      "requiredColumns": [
        "Статус заказа",
        "Стоимость заказа без НДС",
        "Дата заказа",
        "ID заявки",
        "Вес заказов",
        "Пройденное расстояние , км",
        "Cтоимость заказа без НДС, (AZ + BA + BG − BB) руб"
      ],
      "statusRule": {"column": "Статус заказа", "value": "Отмена", "costCondition": ">0"},
      "dateColumn": "Дата заказа",
      "statusCheck": true
    },
    // ... остальные 9 СД
  },
  "emailMapping": {
    "Почта России": {
      "cdEmails": ["Svetlana.Kostenko@russianpost.ru"],
      "avitoEmails": ["benazaryan@avito.ru", "megolubeva@avito.ru"]
    },
    // ... остальные СД
  },
  "storage": {
    "inputPath": "/data/files/input/",
    "outputPath": "/data/files/output/",
    "dbConnection": "postgresql://user:pass@host:5432/rpa_db"
  },
  "integrations": {
    "mattermost": {
      "webhookUrl": "https://mattermost.avito.ru/hooks/xxx",
      "apiUrl": "https://mattermost.avito.ru/api/v4"
    },
    "jupyter": {
      "baseUrl": "https://jupyter.avito.ru",
      "notebookPath": "/notebooks/fincheck_validation.ipynb"
    },
    "outlook": {
      "smtpHost": "smtp.office365.com",
      "smtpPort": 587
    }
  }
}
```

---

## 5. Обработка ошибок в n8n

### 5.1 Стратегия обработки ошибок

```
┌─────────────────────────────────────────────────────────────┐
│                    Error Handling Flow                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Workflow Execution                                          │
│       │                                                      │
│       ▼                                                      │
│  ┌─────────────┐    ┌──────────────┐    ┌──────────────┐   │
│  │ Main Flow   │───▶│ Error Trigger│───▶│ Log Error    │   │
│  │             │ ❌ │  (Node)      │    │ (Grafana/    │   │
│  └─────────────┘    └──────────────┘    │  Redash)     │   │
│                          │               └──────────────┘   │
│                          ▼                                   │
│                   ┌──────────────┐                          │
│                   │ Retry Logic  │ (3 попытки с экспоненци-  │
│                   │ (Code Node)  │  альной задержкой)        │
│                   └──────────────┘                          │
│                          │                                   │
│           ┌──────────────┴──────────────┐                   │
│           ▼                             ▼                   │
│     ┌───────────┐                 ┌───────────┐            │
│     │ Success   │                 │ Max Retry │            │
│     │ Continue  │                 │ Exceeded  │            │
│     └───────────┘                 └───────────┘            │
│                                      │                      │
│                                      ▼                      │
│                               ┌──────────────┐             │
│                               │ Send Alert   │             │
│                               │ (MM Channel) │             │
│                               └──────────────┘             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 Типы ошибок

| Тип ошибки | Код | Обработка | Уведомление |
|------------|-----|-----------|-------------|
| **SE** (System Error) | SE-001, SE-002... | Retry (3x) → Alert | MM канал поддержки |
| **BE** (Business Error) | BE-001, BE-002... | Без retry, сразу отчёт | Инициатор + СД |
| **BE2** (No Data) | BE2-001 | Завершение Success | Инициатор |

---

## 6. Миграция специфичных функций Primo → n8n

### 6.1 Работа с очередями

**Primo**: Встроенная очередь Orchestrator  
**n8n**: 
- Вариант 1: **Wait Node** + **Database** для хранения состояния
- Вариант 2: **External Queue** (Redis/RabbitMQ) через HTTP Request
- Вариант 3: **PostgreSQL с status flag** (рекомендуется)

```sql
CREATE TABLE process_queue (
    id SERIAL PRIMARY KEY,
    request_id VARCHAR(50),
    delivery_service VARCHAR(50),
    status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT NOW(),
    processed_at TIMESTAMP,
    error_message TEXT
);
```

### 6.2 Инициализация

**Primo**: Классический блок инициализации  
**n8n**: Code Node с загрузкой конфигурации

```javascript
// Code Node: Init Config
const fs = require('fs');
const configPath = '/data/config/fincheck_config.json';

try {
  const configData = fs.readFileSync(configPath, 'utf8');
  const config = JSON.parse(configData);
  
  return {
    json: {
      config: config,
      timestamp: new Date().toISOString(),
      requestId: $('Webhook').first().json.requestId
    }
  };
} catch (error) {
  throw new Error(`SE-001: Failed to load config. ${error.message}`);
}
```

### 6.3 Работа с Excel

**Primo**: Встроенные активности Excel  
**n8n**: Code Node + библиотека `xlsx`

```javascript
// Code Node: Validate Excel Columns
const XLSX = require('xlsx');
const fs = require('fs');

const filePath = this.getInputData()[0].binary.file.path;
const workbook = XLSX.readFile(filePath);
const requiredColumns = $('Init Config').first().json.config.deliveryServices[cdName].requiredColumns;

const errors = [];

for (const sheetName of workbook.SheetNames) {
  const worksheet = workbook.Sheets[sheetName];
  const headers = XLSX.utils.sheet_to_json(worksheet, {header: 1})[0];
  
  const missingColumns = requiredColumns.filter(col => 
    !headers.some(h => h.trim().toLowerCase() === col.trim().toLowerCase())
  );
  
  if (missingColumns.length > 0) {
    errors.push({
      service: cdName,
      error: `Отсутствуют столбцы: ${missingColumns.join(', ')}`,
      fileName: filePath,
      sheetName: sheetName
    });
  }
}

return { json: { errors, isValid: errors.length === 0 } };
```

### 6.4 Интеграция с Jupyter Notebook

**Primo**: Специфичный коннектор  
**n8n**: HTTP Request Node к Jupyter API

```javascript
// Подготовка запроса к Jupyter
const jupyterPayload = {
  notebook_path: config.jupyter.notebookPath,
  parameters: {
    delivery_service: cdName,
    request_id: requestId,
    data_source: 'rpa_db'
  }
};

// HTTP Request Node
{
  method: 'POST',
  url: '{{ $json.config.integrations.jupyter.baseUrl }}/api/papermill/execute',
  body: jupyterPayload,
  options: {
    timeout: 300000 // 5 минут на выполнение
  }
}
```

---

## 7. Логирование и мониторинг

### 7.1 Структура логов

```json
{
  "timestamp": "2026-02-16T10:15:25Z",
  "workflow": "FinCheck3PL_Main",
  "executionId": "exec_12345",
  "requestId": "req_67890",
  "deliveryService": "Яндекс ПВЗ",
  "step": "Validate Columns",
  "status": "success",
  "details": {
    "filesProcessed": 2,
    "errorsFound": 0
  },
  "duration": 1250
}
```

### 7.2 Интеграция с Grafana/Redash

- **Вариант 1**: Прямая запись в PostgreSQL → Grafana читает из БД
- **Вариант 2**: HTTP Request к Loki/Prometheus
- **Вариант 3**: Webhook в систему мониторинга

---

## 8. Масштабируемость и производительность

### 8.1 Параллельная обработка

n8n поддерживает параллельное выполнение через:
- **Split In Batches Node**: Обработка файлов пакетами
- **Merge Node**: Агрегация результатов
- **Multiple Workflow Executions**: Параллельный запуск для разных СД

### 8.2 Ограничения n8n

| Параметр | Значение | Рекомендация |
|----------|----------|--------------|
| Макс. время выполнения workflow | Зависит от инфраструктуры | Разбивать на под-workflow |
| Память на execution | ~256MB (по умолчанию) | Использовать стриминг для больших файлов |
| Кол-во concurrent executions | Зависит от лицензии | Настроить queue management |

---

## 9. Требования к инфраструктуре

### 9.1 Минимальная конфигурация сервера n8n

```yaml
version: '3.8'
services:
  n8n:
    image: n8nio/n8n:latest
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=n8n.avito.ru
      - N8N_PROTOCOL=https
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=${DB_PASSWORD}
      - EXECUTIONS_DATA_PRUNE=true
      - EXECUTIONS_DATA_MAX_AGE=168 # часов
    volumes:
      - n8n_data:/home/node/.n8n
      - ./config:/data/config
      - ./files:/data/files
  
  postgres:
    image: postgres:15
    environment:
      - POSTGRES_DB=n8n
      - POSTGRES_USER=n8n
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  n8n_data:
  postgres_data:
```

### 9.2 Сетевая папка для файлов

```bash
# Структура директорий
/data/
├── config/
│   └── fincheck_config.json
├── files/
│   ├── input/          # Входные файлы от пользователей
│   ├── output/         # Отчёты с ошибками
│   ├── temp/           # Временные файлы для обработки
│   └── archive/        # Архив обработанных файлов
└── logs/
    └── executions/     # Логи выполнения
```

---

## 10. План миграции

### Этап 1: Подготовка (1 неделя)
- [ ] Развёртывание n8n инфраструктуры
- [ ] Настройка PostgreSQL базы данных
- [ ] Создание конфигурационных файлов для 11 СД
- [ ] Настройка интеграций (Mattermost, Outlook, Jupyter)

### Этап 2: Разработка Workflow 1 (2 недели)
- [ ] Реализация триггера от Mattermost
- [ ] Разработка модуля валидации Excel (11 СД)
- [ ] Интеграция с БД для хранения данных
- [ ] Реализация обработки ошибок

### Этап 3: Разработка Workflow 2 (2 недели)
- [ ] Реализация проверок (статусы, даты, веса)
- [ ] Интеграция с Jupyter Notebook
- [ ] Генерация отчётов в Excel
- [ ] Настройка уведомлений (MM + Email)

### Этап 4: Тестирование (1 неделя)
- [ ] Unit-тесты для каждого узла
- [ ] Интеграционные тесты с реальными файлами
- [ ] Нагрузочное тестирование (21 файл/мес)
- [ ] User Acceptance Testing с логистами

### Этап 5: Промышленная эксплуатация (ongoing)
- [ ] Мониторинг через Grafana
- [ ] Настройка алертов
- [ ] Документирование операций
- [ ] Обучение команды поддержки

---

## 11. Риски и рекомендации

### Риски

| Риск | Вероятность | Влияние | Митигация |
|------|-------------|---------|-----------|
| Отсутствие готовых коннекторов для 1С/DWH | Средняя | Высокое | Разработка кастомных HTTP Request nodes |
| Ограничения n8n по работе с большими Excel | Низкая | Среднее | Стриминговая обработка, разбивка на части |
| Сложность отладки распределённых workflow | Средняя | Среднее | Детальное логирование, тестовые окружения |
| Зависимость от внешних API (Jupyter, MM) | Высокая | Высокое | Retry logic, circuit breaker pattern |

### Рекомендации

1. **Использовать version control** для workflow (экспорт в JSON + Git)
2. **Создать тестовые данные** для каждой из 11 СД
3. **Реализовать feature flags** в конфиге для постепенного включения СД
4. **Настроить staging окружение** для тестирования перед prod
5. **Документировать каждый Code Node** с примерами входных/выходных данных

---

## 12. Заключение

Переход с **Primo RPA** на **n8n** требует значительной переработки архитектуры процесса, но предоставляет следующие преимущества:

✅ **Гибкость**: Легче добавлять новые СД через конфигурацию  
✅ **Прозрачность**: Визуальное представление workflow  
✅ **Интеграции**: Богатая экосистема готовых коннекторов  
✅ **Масштабируемость**: Горизонтальное масштабирование через Docker  
✅ **Cost-effective**: Open-source версия доступна бесплатно  

**Ориентировочная трудоёмкость**: 6-8 недель (с учётом тестирования и документации)

**Критические зависимости**:
- Наличие PostgreSQL базы данных
- Доступ к API Mattermost и Jupyter
- Настроенный SMTP сервер для Outlook
- Сетевое хранилище для файлов

---

*Документ подготовлен для согласования архитектуры перед началом разработки.*

*Версия: 1.0*  
*Дата: 2026-02-16*
