# Workflow 1: FinCheck3PL_Trigger (Основной workflow)

## 1. Общая информация

| Параметр | Значение |
|----------|----------|
| **Название** | FinCheck3PL_Trigger |
| **Тип** | Основной workflow |
| **Триггер** | Webhook от Mattermost |
| **Версия** | 1.0 |
| **Статус** | Разработка |
| **Зависимости** | PostgreSQL, Mattermost API, Outlook SMTP, Jupyter API |

---

## 2. Архитектура Workflow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    WORKFLOW: FinCheck3PL_Trigger                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────┐                                                       │
│  │   START      │                                                       │
│  │  (Webhook)   │                                                       │
│  └──────┬───────┘                                                       │
│         │                                                                │
│         ▼                                                                │
│  ┌──────────────┐                                                       │
│  │   Node 1     │ ◀── Инициализация (загрузка конфига)                  │
│  │  Init Config │                                                       │
│  └──────┬───────┘                                                       │
│         │                                                                │
│         ▼                                                                │
│  ┌──────────────┐                                                       │
│  │   Node 2     │ ◀── Получение файлов из webhook                       │
│  │  Get Files   │     Сохранение в /data/files/input/                   │
│  └──────┬───────┘                                                       │
│         │                                                                │
│         ▼                                                                │
│  ┌──────────────┐                                                       │
│  │   Node 3     │ ◀── Определение СД (из dropdown MM)                   │
│  │ Identify CD  │                                                       │
│  └──────┬───────┘                                                       │
│         │                                                                │
│         ▼                                                                │
│  ┌──────────────┐                                                       │
│  │   Node 4     │ ◀── Проверка обязательных колонок/листов              │
│  │  Validate    │     Для каждой СД свой набор                          │
│  │   Columns    │                                                       │
│  └──────┬───────┘                                                       │
│         │                                                                │
│         ├─────────────────┐                                              │
│         │                 │                                              │
│         ▼ (Error)         ▼ (Success)                                    │
│  ┌──────────────┐   ┌──────────────┐                                    │
│  │   Node 5E    │   │   Node 5S    │                                    │
│  │ Send Error   │   │ Save to DB   │                                    │
│  │ Notification │   │ (PostgreSQL) │                                    │
│  └──────┬───────┘   └──────┬───────┘                                    │
│         │                  │                                            │
│         │                  ▼                                            │
│         │           ┌──────────────┐                                    │
│         │           │   Node 6     │                                    │
│         │           │ Check Data   │ ◀── Есть ли данные?                │
│         │           └──────┬───────┘                                    │
│         │                  │                                            │
│         │         ┌────────┴────────┐                                   │
│         │         │                 │                                   │
│         │         ▼ (No Data)       ▼ (Has Data)                        │
│         │  ┌──────────────┐  ┌──────────────┐                           │
│         │  │   Node 7A    │  │   Node 7B    │                           │
│         │  │ BE2 Complete │  │ Call WFlow 2 │                           │
│         │  │ (Success)    │  │ (Jupyter)    │                           │
│         │  └──────┬───────┘  └──────┬───────┘                           │
│         │         │                 │                                    │
│         │         │                 ▼                                    │
│         │         │        ┌──────────────┐                             │
│         │         │        │   Node 8     │                             │
│         │         │        │ Get Results  │                             │
│         │         │        └──────┬───────┘                             │
│         │         │               │                                     │
│         │         │               ▼                                     │
│         │         │        ┌──────────────┐                             │
│         │         │        │   Node 9     │                             │
│         │         │        │ Generate     │                             │
│         │         │        │ Report Excel │                             │
│         │         │        └──────┬───────┘                             │
│         │         │               │                                     │
│         │         │               ▼                                     │
│         └─────────┴───────────────┼─────────────────────────────────────┘
│                                   │
│                                   ▼
│                            ┌──────────────┐
│                            │   Node 10    │
│                            │ Send Notify  │ ◀── MM + Outlook
│                            └──────┬───────┘
│                                   │
│                                   ▼
│                            ┌──────────────┐
│                            │    END       │
│                            └──────────────┘
│
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Детальное описание узлов (Nodes)

### Node 0: Webhook Trigger

**Тип:** `n8n-nodes-base.webhook`

**Конфигурация:**
```json
{
  "httpMethod": "POST",
  "path": "fincheck3pl-trigger",
  "authentication": "headerAuth",
  "responseMode": "lastNode",
  "options": {
    "binaryData": true,
    "rawBody": false
  }
}
```

**Входные данные от Mattermost:**
```json
{
  "requestId": "req_{{timestamp}}",
  "initiator": "user_id_123",
  "channelId": "channel_456",
  "deliveryService": "5post",
  "comment": "Проверка файла за январь 2026",
  "files": [
    {
      "name": "5post_january_2026.xlsx",
      "mimeType": "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
      "size": 245678
    }
  ]
}
```

---

### Node 1: Init Config

**Тип:** `n8n-nodes-base.code`

**Описание:** Загрузка конфигурационного файла с настройками для всех СД

**Code (JavaScript):**
```javascript
const fs = require('fs');

// Путь к конфигурационному файлу
const configPath = '/data/config/fincheck_config.json';

try {
  // Чтение конфигурации
  const configData = fs.readFileSync(configPath, 'utf8');
  const config = JSON.parse(configData);
  
  // Извлечение данных из webhook
  const webhookData = $input.first().json;
  
  // Формирование контекста выполнения
  const executionContext = {
    requestId: webhookData.requestId || `req_${Date.now()}`,
    initiator: webhookData.initiator,
    channelId: webhookData.channelId,
    comment: webhookData.comment,
    deliveryService: webhookData.deliveryService,
    timestamp: new Date().toISOString(),
    config: config,
    status: 'initialized',
    errors: [],
    warnings: []
  };
  
  return {
    json: executionContext
  };
  
} catch (error) {
  // Логирование ошибки инициализации
  const errorResponse = {
    json: {
      requestId: `req_${Date.now()}`,
      status: 'error',
      errorCode: 'SE-001',
      errorMessage: `Failed to load config: ${error.message}`,
      timestamp: new Date().toISOString()
    }
  };
  
  throw new Error(`SE-001: ${error.message}`);
}
```

**Выходные данные:**
```json
{
  "requestId": "req_1708081234567",
  "initiator": "user_id_123",
  "channelId": "channel_456",
  "comment": "Проверка файла за январь 2026",
  "deliveryService": "5post",
  "timestamp": "2026-02-16T10:15:25Z",
  "config": { ... },
  "status": "initialized",
  "errors": [],
  "warnings": []
}
```

---

### Node 2: Get Files

**Тип:** `n8n-nodes-base.code`

**Описание:** Обработка и сохранение файлов из webhook во временное хранилище

**Code (JavaScript):**
```javascript
const fs = require('fs');
const path = require('path');

// Получение бинарных данных из webhook
const inputData = $input.first();
const context = inputData.json;
const binaryFiles = inputData.binary;

const inputPath = context.config.storage.inputPath;
const requestId = context.requestId;

// Создание директории для запроса
const requestDir = path.join(inputPath, requestId);
if (!fs.existsSync(requestDir)) {
  fs.mkdirSync(requestDir, { recursive: true });
}

const savedFiles = [];
const fileErrors = [];

// Обработка каждого файла
if (binaryFiles && Object.keys(binaryFiles).length > 0) {
  for (const [key, file] of Object.entries(binaryFiles)) {
    try {
      // Генерация уникального имени файла
      const originalName = file.fileName || `file_${Date.now()}.xlsx`;
      const safeName = originalName.replace(/[^a-zA-Z0-9._-]/g, '_');
      const filePath = path.join(requestDir, safeName);
      
      // Сохранение файла
      fs.writeFileSync(filePath, file.data);
      
      savedFiles.push({
        originalName: originalName,
        savedName: safeName,
        fullPath: filePath,
        size: file.fileSize || fs.statSync(filePath).size,
        mimeType: file.mimeType || 'application/octet-stream',
        uploadedAt: new Date().toISOString()
      });
      
    } catch (error) {
      fileErrors.push({
        fileName: binaryFiles[key]?.fileName || 'unknown',
        error: error.message
      });
    }
  }
}

// Проверка наличия файлов
if (savedFiles.length === 0 && fileErrors.length === 0) {
  throw new Error('BE-001: No files attached to the request');
}

// Обновление контекста
context.files = savedFiles;
context.fileErrors = fileErrors;
context.status = 'files_received';

return {
  json: context,
  binary: {}
};
```

**Выходные данные:**
```json
{
  "requestId": "req_1708081234567",
  "deliveryService": "5post",
  "files": [
    {
      "originalName": "5post_january_2026.xlsx",
      "savedName": "5post_january_2026.xlsx",
      "fullPath": "/data/files/input/req_1708081234567/5post_january_2026.xlsx",
      "size": 245678,
      "mimeType": "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
      "uploadedAt": "2026-02-16T10:15:26Z"
    }
  ],
  "status": "files_received"
}
```

---

### Node 3: Identify CD (Delivery Service)

**Тип:** `n8n-nodes-base.switch`

**Описание:** Определение службы доставки и выбор соответствующей конфигурации

**Конфигурация:**
```json
{
  "dataType": "string",
  "value1": "={{ $json.deliveryService }}",
  "rules": [
    {
      "value2": "5post",
      "output": 0
    },
    {
      "value2": "yandexCity",
      "output": 1
    },
    {
      "value2": "yandexPVZ",
      "output": 2
    },
    {
      "value2": "dostavista",
      "output": 3
    },
    {
      "value2": "pochtaRossii",
      "output": 4
    },
    {
      "value2": "cdek",
      "output": 5
    },
    {
      "value2": "boxberry",
      "output": 6
    },
    {
      "value2": "dpd",
      "output": 7
    },
    {
      "value2": "kurier",
      "output": 8
    },
    {
      "value2": "ems",
      "output": 9
    },
    {
      "value2": "other",
      "output": 10
    }
  ],
  "fallbackOutput": 10
}
```

**Логика:**
- Вход: название СД из webhook (`deliveryService`)
- Выход: индекс соответствующей ветки обработки
- Fallback: обработка как "other" с ошибкой валидации

---

### Node 4: Validate Columns

**Тип:** `n8n-nodes-base.code`

**Описание:** Проверка наличия обязательных колонок и листов в Excel файле

**Code (JavaScript):**
```javascript
const XLSX = require('xlsx');
const fs = require('fs');
const path = require('path');

const context = $input.first().json;
const cdName = context.deliveryService;
const files = context.files;

// Получение конфигурации для текущей СД
const cdConfig = context.config.deliveryServices[cdName];

if (!cdConfig) {
  throw new Error(`BE-002: Configuration not found for delivery service: ${cdName}`);
}

const requiredColumns = cdConfig.requiredColumns || [];
const requiredSheets = cdConfig.requiredSheets || [];
const weightLimit = cdConfig.weightLimit || 0;

const validationResults = [];
const allErrors = [];

// Обработка каждого файла
for (const file of files) {
  const filePath = file.fullPath;
  
  try {
    // Чтение Excel файла
    const workbook = XLSX.readFile(filePath);
    const sheetNames = workbook.SheetNames;
    
    const fileResult = {
      fileName: file.originalName,
      filePath: filePath,
      sheets: [],
      errors: [],
      isValid: true
    };
    
    // Проверка обязательных листов (если указано в конфиге)
    if (requiredSheets.length > 0) {
      const missingSheets = requiredSheets.filter(sheet => 
        !sheetNames.some(s => s.toLowerCase() === sheet.toLowerCase())
      );
      
      if (missingSheets.length > 0) {
        const error = {
          type: 'MISSING_SHEETS',
          message: `Отсутствуют обязательные листы: ${missingSheets.join(', ')}`,
          code: 'BE-003'
        };
        fileResult.errors.push(error);
        allErrors.push({ ...error, fileName: file.originalName });
        fileResult.isValid = false;
      }
    }
    
    // Проверка каждого листа
    for (const sheetName of sheetNames) {
      const worksheet = workbook.Sheets[sheetName];
      
      // Получение заголовков (первая строка)
      const headers = XLSX.utils.sheet_to_json(worksheet, { header: 1 })[0] || [];
      const normalizedHeaders = headers.map(h => String(h).trim().toLowerCase());
      
      // Проверка обязательных колонок
      const missingColumns = requiredColumns.filter(col => {
        const normalizedCol = col.trim().toLowerCase();
        return !normalizedHeaders.includes(normalizedCol);
      });
      
      const sheetResult = {
        sheetName: sheetName,
        rowCount: XLSX.utils.sheet_to_json(worksheet, { header: 1 }).length - 1,
        columns: headers,
        missingColumns: missingColumns,
        isValid: missingColumns.length === 0
      };
      
      fileResult.sheets.push(sheetResult);
      
      if (missingColumns.length > 0) {
        const error = {
          type: 'MISSING_COLUMNS',
          sheetName: sheetName,
          message: `Отсутствуют столбцы: ${missingColumns.join(', ')}`,
          code: 'BE-004',
          missingColumns: missingColumns
        };
        fileResult.errors.push(error);
        allErrors.push({ ...error, fileName: file.originalName });
        fileResult.isValid = false;
      }
      
      // Дополнительная проверка: количество строк
      if (sheetResult.rowCount === 0) {
        const warning = {
          type: 'EMPTY_SHEET',
          sheetName: sheetName,
          message: 'Лист не содержит данных',
          code: 'BE-005'
        };
        fileResult.errors.push(warning);
        context.warnings.push({ ...warning, fileName: file.originalName });
      }
    }
    
    validationResults.push(fileResult);
    
  } catch (error) {
    const fileError = {
      fileName: file.originalName,
      type: 'FILE_READ_ERROR',
      message: `Ошибка чтения файла: ${error.message}`,
      code: 'SE-002'
    };
    validationResults.push({
      fileName: file.originalName,
      isValid: false,
      errors: [fileError]
    });
    allErrors.push(fileError);
  }
}

// Формирование итогового результата
const isAllValid = validationResults.every(r => r.isValid);

context.validationResults = validationResults;
context.errors = [...context.errors, ...allErrors];
context.isValid = isAllValid;
context.status = isAllValid ? 'validated' : 'validation_failed';

return {
  json: context
};
```

**Выходные данные (успех):**
```json
{
  "requestId": "req_1708081234567",
  "deliveryService": "5post",
  "isValid": true,
  "status": "validated",
  "validationResults": [
    {
      "fileName": "5post_january_2026.xlsx",
      "sheets": [
        {
          "sheetName": "Sheet1",
          "rowCount": 1250,
          "columns": ["Дата и время...", "Физический вес...", ...],
          "missingColumns": [],
          "isValid": true
        }
      ],
      "errors": [],
      "isValid": true
    }
  ],
  "errors": [],
  "warnings": []
}
```

**Выходные данные (ошибка):**
```json
{
  "requestId": "req_1708081234567",
  "deliveryService": "5post",
  "isValid": false,
  "status": "validation_failed",
  "validationResults": [
    {
      "fileName": "5post_january_2026.xlsx",
      "sheets": [
        {
          "sheetName": "Sheet1",
          "rowCount": 1250,
          "columns": ["Дата", "Вес"],
          "missingColumns": ["Стоимость за доставку...", "Итоговая стоимость..."],
          "isValid": false
        }
      ],
      "errors": [
        {
          "type": "MISSING_COLUMNS",
          "sheetName": "Sheet1",
          "message": "Отсутствуют столбцы: Стоимость за доставку..., Итоговая стоимость...",
          "code": "BE-004"
        }
      ],
      "isValid": false
    }
  ],
  "errors": [...],
  "warnings": []
}
```

---

### Node 5: IF Node (Validation Check)

**Тип:** `n8n-nodes-base.if`

**Конфигурация:**
```json
{
  "conditions": {
    "boolean": [
      {
        "value1": "={{ $json.isValid }}",
        "operation": "equal",
        "value2": true
      }
    ]
  }
}
```

**Логика:**
- **True branch** → Node 6 (Save to DB)
- **False branch** → Node 5E (Send Error Notification)

---

### Node 5E: Send Error Notification (Business Error)

**Тип:** `n8n-nodes-base.code` + `n8n-nodes-base.httpRequest` + `n8n-nodes-base.emailSend`

**Описание:** Формирование и отправка уведомления об ошибках валидации

**Code (JavaScript) - Подготовка сообщения:**
```javascript
const context = $input.first().json;
const errors = context.errors || [];
const cdName = context.deliveryService;

// Формирование текста сообщения
let mmMessage = `❌ **FinCheck3PL: Ошибки валидации**\n\n`;
mmMessage += `📋 **Запрос:** ${context.requestId}\n`;
mmMessage += `🚚 **Служба доставки:** ${cdName}\n`;
mmMessage += `👤 **Инициатор:** ${context.initiator}\n`;
mmMessage += `📁 **Файлов:** ${context.files?.length || 0}\n\n`;

mmMessage += `⚠️ **Обнаружены ошибки:**\n`;

for (const error of errors) {
  mmMessage += `\n• **Файл:** ${error.fileName}\n`;
  mmMessage += `  • **Тип:** ${error.type}\n`;
  mmMessage += `  • **Сообщение:** ${error.message}\n`;
  if (error.sheetName) {
    mmMessage += `  • **Лист:** ${error.sheetName}\n`;
  }
}

mmMessage += `\n💡 **Рекомендация:** Проверьте формат файла согласно требованиям для ${cdName}`;

// Получение email получателей
const emailMapping = context.config.emailMapping[cdName] || {};
const initiatorEmails = emailMapping.avitoEmails || ['rpa-support@avito.ru'];

const emailSubject = `[FinCheck3PL] Ошибки валидации: ${context.requestId}`;
const emailBody = `
<html>
<body>
  <h2>❌ FinCheck3PL: Ошибки валидации</h2>
  <table border="1" cellpadding="5">
    <tr><th>Параметр</th><th>Значение</th></tr>
    <tr><td>Запрос</td><td>${context.requestId}</td></tr>
    <tr><td>Служба доставки</td><td>${cdName}</td></tr>
    <tr><td>Инициатор</td><td>${context.initiator}</td></tr>
    <tr><td>Файлов</td><td>${context.files?.length || 0}</td></tr>
  </table>
  
  <h3>Обнаруженные ошибки:</h3>
  <ul>
    ${errors.map(e => `
      <li>
        <strong>Файл:</strong> ${e.fileName}<br>
        <strong>Тип:</strong> ${e.type}<br>
        <strong>Сообщение:</strong> ${e.message}<br>
        ${e.sheetName ? `<strong>Лист:</strong> ${e.sheetName}<br>` : ''}
      </li>
    `).join('')}
  </ul>
  
  <p><strong>Рекомендация:</strong> Проверьте формат файла согласно требованиям для ${cdName}</p>
</body>
</html>
`;

return {
  json: {
    ...context,
    mmMessage: mmMessage,
    emailSubject: emailSubject,
    emailBody: emailBody,
    recipientEmails: initiatorEmails,
    notificationType: 'validation_error'
  }
};
```

**HTTP Request Node (Mattermost):**
```json
{
  "method": "POST",
  "url": "={{ $json.config.integrations.mattermost.webhookUrl }}",
  "sendBody": true,
  "bodyParameters": [
    {
      "parameter": "text",
      "value": "={{ $json.mmMessage }}"
    },
    {
      "parameter": "channel",
      "value": "={{ $json.channelId }}"
    },
    {
      "parameter": "username",
      "value": "FinCheck3PL Bot"
    }
  ]
}
```

**SMTP Node (Outlook):**
```json
{
  "from": "rpa-fincheck@avito.ru",
  "to": "={{ $json.recipientEmails.join(';') }}",
  "subject": "={{ $json.emailSubject }}",
  "emailType": "html",
  "message": "={{ $json.emailBody }}"
}
```

---

### Node 6: Save to DB

**Тип:** `n8n-nodes-base.postgres`

**Описание:** Сохранение данных из Excel файлов в PostgreSQL базу данных

**Code (JavaScript) - Подготовка данных:**
```javascript
const XLSX = require('xlsx');
const fs = require('fs');

const context = $input.first().json;
const files = context.files;
const cdName = context.deliveryService;
const requestId = context.requestId;
const timestamp = context.timestamp;

const recordsToInsert = [];
let totalRows = 0;

// Обработка каждого файла
for (const file of files) {
  const filePath = file.fullPath;
  const workbook = XLSX.readFile(filePath);
  
  // Обработка каждого листа
  for (const sheetName of workbook.SheetNames) {
    const worksheet = workbook.Sheets[sheetName];
    
    // Конвертация в JSON с заголовками
    const rows = XLSX.utils.sheet_to_json(worksheet);
    
    for (const row of rows) {
      // Нормализация данных
      const record = {
        request_id: requestId,
        delivery_service: cdName,
        file_name: file.originalName,
        sheet_name: sheetName,
        created_at: timestamp,
        status: 'pending',
        is_filled_in_lines: false,
        is_checking_for_statuses: false,
        is_checking_for_date: false,
        is_checking_for_weights: false,
        jupyter_checked: false,
        ...row // Данные из строки Excel
      };
      
      recordsToInsert.push(record);
      totalRows++;
    }
  }
}

context.dbRecordsCount = totalRows;
context.recordsToInsert = recordsToInsert;
context.status = 'ready_for_db';

return {
  json: context
};
```

**PostgreSQL Node - Конфигурация:**
```json
{
  "operation": "executeQuery",
  "query": "INSERT INTO fincheck_orders (\n    request_id, delivery_service, file_name, sheet_name,\n    created_at, status, is_filled_in_lines,\n    is_checking_for_statuses, is_checking_for_date,\n    is_checking_for_weights, jupyter_checked, data\n  ) VALUES \n  {{ $records.map((record, index) => \n    `($${index * 10 + 1}, $${index * 10 + 2}, $${index * 10 + 3}, $${index * 10 + 4}, $${index * 10 + 5}, $${index * 10 + 6}, $${index * 10 + 7}, $${index * 10 + 8}, $${index * 10 + 9}, $${index * 10 + 10}, $${index * 10 + 11}, $${index * 10 + 12})`\n  ).join(', ') }}\n  ON CONFLICT (request_id, delivery_service) DO NOTHING",
  "values": "={{ $json.recordsToInsert.flatMap(r => [\n    r.request_id, r.delivery_service, r.file_name, r.sheet_name,\n    r.created_at, r.status, r.is_filled_in_lines,\n    r.is_checking_for_statuses, r.is_checking_for_date,\n    r.is_checking_for_weights, r.jupyter_checked,\n    JSON.stringify(r)\n  ]) }}"
}
```

**SQL таблица (создание):**
```sql
CREATE TABLE IF NOT EXISTS fincheck_orders (
    id SERIAL PRIMARY KEY,
    request_id VARCHAR(50) NOT NULL,
    delivery_service VARCHAR(50) NOT NULL,
    file_name VARCHAR(255),
    sheet_name VARCHAR(100),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    status VARCHAR(20) DEFAULT 'pending',
    is_filled_in_lines BOOLEAN DEFAULT FALSE,
    is_checking_for_statuses BOOLEAN DEFAULT FALSE,
    is_checking_for_date BOOLEAN DEFAULT FALSE,
    is_checking_for_weights BOOLEAN DEFAULT FALSE,
    jupyter_checked BOOLEAN DEFAULT FALSE,
    data JSONB,
    UNIQUE(request_id, delivery_service, file_name, sheet_name)
);

CREATE INDEX idx_fincheck_request_id ON fincheck_orders(request_id);
CREATE INDEX idx_fincheck_delivery_service ON fincheck_orders(delivery_service);
CREATE INDEX idx_fincheck_status ON fincheck_orders(status);
CREATE INDEX idx_fincheck_created_at ON fincheck_orders(created_at);
```

---

### Node 7: Check Data (IF Node)

**Тип:** `n8n-nodes-base.if`

**Конфигурация:**
```json
{
  "conditions": {
    "number": [
      {
        "value1": "={{ $json.dbRecordsCount }}",
        "operation": "larger",
        "value2": 0
      }
    ]
  }
}
```

**Логика:**
- **True branch** (есть данные) → Node 7B (Call Workflow 2)
- **False branch** (нет данных) → Node 7A (BE2 Complete)

---

### Node 7A: BE2 Complete (No Data)

**Тип:** `n8n-nodes-base.code`

**Описание:** Завершение процесса без данных (не является ошибкой)

**Code (JavaScript):**
```javascript
const context = $input.first().json;

context.status = 'completed_no_data';
context.errorCode = 'BE2-001';
context.errorMessage = 'Нет данных для обработки после валидации';

// Формирование уведомления
context.mmMessage = `⚠️ **FinCheck3PL: Нет данных**\n\n`;
context.mmMessage += `📋 **Запрос:** ${context.requestId}\n`;
context.mmMessage += `🚚 **Служба доставки:** ${context.deliveryService}\n`;
context.mmMessage += `👤 **Инициатор:** ${context.initiator}\n\n`;
context.mmMessage += `ℹ️ **Статус:** Файлы получены, но не содержат данных для обработки.\n`;
context.mmMessage += `Возможно, файл пуст или содержит только заголовки.`;

context.emailSubject = `[FinCheck3PL] Нет данных: ${context.requestId}`;
context.emailBody = `
<html>
<body>
  <h2>⚠️ FinCheck3PL: Нет данных для обработки</h2>
  <table border="1" cellpadding="5">
    <tr><th>Параметр</th><th>Значение</th></tr>
    <tr><td>Запрос</td><td>${context.requestId}</td></tr>
    <tr><td>Служба доставки</td><td>${context.deliveryService}</td></tr>
    <tr><td>Инициатор</td><td>${context.initiator}</td></tr>
    <tr><td>Статус</td><td>completed_no_data</td></tr>
  </table>
  
  <p><strong>Сообщение:</strong> Файлы получены, но не содержат данных для обработки.</p>
  <p>Возможно, файл пуст или содержит только заголовки.</p>
</body>
</html>
`;

context.notificationType = 'no_data';

return {
  json: context
};
```

---

### Node 7B: Call Workflow 2 (Jupyter Integration)

**Тип:** `n8n-nodes-base.executeWorkflow`

**Описание:** Запуск Workflow 2 для глубокой проверки данных через Jupyter Notebook

**Конфигурация:**
```json
{
  "workflowId": "FinCheck3PL_JupyterCheck",
  "source": "node",
  "inputs": {
    "main": [
      [
        {
          "json": {
            "requestId": "={{ $json.requestId }}",
            "deliveryService": "={{ $json.deliveryService }}",
            "timestamp": "={{ $json.timestamp }}",
            "config": "={{ $json.config }}",
            "dbRecordsCount": "={{ $json.dbRecordsCount }}",
            "initiator": "={{ $json.initiator }}",
            "channelId": "={{ $json.channelId }}"
          }
        }
      ]
    ]
  }
}
```

**Ожидаемые выходные данные от Workflow 2:**
```json
{
  "requestId": "req_1708081234567",
  "status": "jupyter_completed",
  "jupyterResults": {
    "totalChecked": 1250,
    "errorsFound": 45,
    "checks": {
      "filledLines": { checked: 1250, errors: 12 },
      "statuses": { checked: 800, errors: 8 },
      "dates": { checked: 1250, errors: 15 },
      "weights": { checked: 1250, errors: 10 }
    }
  },
  "reportFile": "/data/files/output/req_1708081234567_report.xlsx"
}
```

---

### Node 8: Get Results

**Тип:** `n8n-nodes-base.code`

**Описание:** Обработка результатов от Workflow 2

**Code (JavaScript):**
```javascript
const workflow2Results = $input.first().json;

// Объединение с текущим контекстом
const context = {
  ...workflow2Results,
  status: 'results_received',
  processedAt: new Date().toISOString()
};

// Логирование результатов
console.log(`Workflow 2 completed for ${context.requestId}`);
console.log(`Total checked: ${context.jupyterResults?.totalChecked || 0}`);
console.log(`Errors found: ${context.jupyterResults?.errorsFound || 0}`);

return {
  json: context
};
```

---

### Node 9: Generate Report Excel

**Тип:** `n8n-nodes-base.code` + `n8n-nodes-base.writeBinaryFile`

**Описание:** Генерация отчёта с ошибками в формате Excel

**Code (JavaScript):**
```javascript
const XLSX = require('xlsx');
const fs = require('fs');
const path = require('path');

const context = $input.first().json;
const requestId = context.requestId;
const cdName = context.deliveryService;
const jupyterResults = context.jupyterResults || {};
const outputPath = context.config.storage.outputPath;

// Создание отчёта
const reportData = [];

// Заголовок отчёта
reportData.push({
  Section: 'SUMMARY',
  Field: 'Request ID',
  Value: requestId
});

reportData.push({
  Section: 'SUMMARY',
  Field: 'Delivery Service',
  Value: cdName
});

reportData.push({
  Section: 'SUMMARY',
  Field: 'Processed At',
  Value: context.processedAt
});

reportData.push({
  Section: 'SUMMARY',
  Field: 'Total Checked',
  Value: jupyterResults.totalChecked || 0
});

reportData.push({
  Section: 'SUMMARY',
  Field: 'Total Errors',
  Value: jupyterResults.errorsFound || 0
});

// Детализация по проверкам
if (jupyterResults.checks) {
  for (const [checkName, checkData] of Object.entries(jupyterResults.checks)) {
    reportData.push({
      Section: 'CHECK_DETAILS',
      Field: checkName,
      Value: `Checked: ${checkData.checked}, Errors: ${checkData.errors}`
    });
  }
}

// Создание workbook
const workbook = XLSX.utils.book_new();
const worksheet = XLSX.utils.json_to_sheet(reportData);

// Добавление листа
XLSX.utils.book_append_sheet(workbook, worksheet, 'Report');

// Сохранение файла
const reportFileName = `${requestId}_report.xlsx`;
const reportFilePath = path.join(outputPath, reportFileName);

// Создание директории если не существует
const outputDir = path.dirname(reportFilePath);
if (!fs.existsSync(outputDir)) {
  fs.mkdirSync(outputDir, { recursive: true });
}

XLSX.writeFile(workbook, reportFilePath);

context.reportFile = reportFilePath;
context.reportFileName = reportFileName;
context.status = 'report_generated';

// Подготовка бинарных данных для attachment
const fileBuffer = fs.readFileSync(reportFilePath);

return {
  json: context,
  binary: {
    report: {
      data: fileBuffer,
      fileName: reportFileName,
      mimeType: 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
    }
  }
};
```

---

### Node 10: Send Notification (Final)

**Тип:** `n8n-nodes-base.code` + `n8n-nodes-base.httpRequest` + `n8n-nodes-base.emailSend`

**Описание:** Отправка финальных уведомлений с отчётом

**Code (JavaScript) - Подготовка сообщения:**
```javascript
const context = $input.first().json;
const cdName = context.deliveryService;
const jupyterResults = context.jupyterResults || {};
const errorsFound = jupyterResults.errorsFound || 0;

// Определение типа уведомления (success vs errors)
const hasErrors = errorsFound > 0;
const statusEmoji = hasErrors ? '⚠️' : '✅';
const statusText = hasErrors ? 'Обнаружены ошибки' : 'Ошибок не найдено';

// Формирование сообщения для Mattermost
let mmMessage = `${statusEmoji} **FinCheck3PL: Проверка завершена**\n\n`;
mmMessage += `📋 **Запрос:** ${context.requestId}\n`;
mmMessage += `🚚 **Служба доставки:** ${cdName}\n`;
mmMessage += `👤 **Инициатор:** ${context.initiator}\n`;
mmMessage += `📊 **Проверено записей:** ${jupyterResults.totalChecked || 0}\n`;
mmMessage += `❌ **Найдено ошибок:** ${errorsFound}\n\n`;

if (hasErrors) {
  mmMessage += `📈 **Детализация проверок:**\n`;
  
  if (jupyterResults.checks) {
    for (const [checkName, checkData] of Object.entries(jupyterResults.checks)) {
      const emoji = checkData.errors > 0 ? '❌' : '✅';
      mmMessage += `${emoji} **${checkName}:** ${checkData.checked} проверено, ${checkData.errors} ошибок\n`;
    }
  }
  
  mmMessage += `\n📎 **Отчёт:** Во вложении`;
} else {
  mmMessage += `✅ Все проверки пройдены успешно!\n`;
  mmMessage += `\n📎 **Отчёт:** Во вложении`;
}

// Получение email получателей
const emailMapping = context.config.emailMapping[cdName] || {};
const cdEmails = emailMapping.cdEmails || [];
const avitoEmails = emailMapping.avitoEmails || [];
const allEmails = [...new Set([...cdEmails, ...avitoEmails])];

const emailSubject = `[FinCheck3PL] ${statusText}: ${context.requestId} (${cdName})`;
const emailBody = `
<html>
<body>
  <h2>${statusEmoji} FinCheck3PL: Проверка завершена</h2>
  
  <table border="1" cellpadding="5">
    <tr><th>Параметр</th><th>Значение</th></tr>
    <tr><td>Запрос</td><td>${context.requestId}</td></tr>
    <tr><td>Служба доставки</td><td>${cdName}</td></tr>
    <tr><td>Инициатор</td><td>${context.initiator}</td></tr>
    <tr><td>Проверено записей</td><td>${jupyterResults.totalChecked || 0}</td></tr>
    <tr><td>Найдено ошибок</td><td style="color: ${hasErrors ? 'red' : 'green'}; font-weight: bold;">${errorsFound}</td></tr>
  </table>
  
  ${hasErrors ? `
    <h3>Детализация проверок:</h3>
    <table border="1" cellpadding="5">
      <tr><th>Проверка</th><th>Проверено</th><th>Ошибок</th></tr>
      ${Object.entries(jupyterResults.checks || {}).map(([name, data]) => `
        <tr>
          <td>${name}</td>
          <td>${data.checked}</td>
          <td style="color: ${data.errors > 0 ? 'red' : 'green'};">${data.errors}</td>
        </tr>
      `).join('')}
    </table>
  ` : '<p><strong>✅ Все проверки пройдены успешно!</strong></p>'}
  
  <p><strong>Отчёт:</strong> См. во вложении</p>
  
  <hr>
  <p style="font-size: 12px; color: gray;">
    Это сообщение автоматически сгенерировано системой FinCheck3PL.<br>
    При возникновении вопросов обратитесь в команду поддержки RPA.
  </p>
</body>
</html>
`;

return {
  json: {
    ...context,
    mmMessage: mmMessage,
    emailSubject: emailSubject,
    emailBody: emailBody,
    recipientEmails: allEmails,
    notificationType: hasErrors ? 'completed_with_errors' : 'completed_success',
    hasErrors: hasErrors
  },
  binary: $input.first().binary
};
```

**HTTP Request Node (Mattermost):**
```json
{
  "method": "POST",
  "url": "={{ $json.config.integrations.mattermost.webhookUrl }}",
  "sendBody": true,
  "bodyParameters": [
    {
      "parameter": "text",
      "value": "={{ $json.mmMessage }}"
    },
    {
      "parameter": "channel",
      "value": "={{ $json.channelId }}"
    },
    {
      "parameter": "username",
      "value": "FinCheck3PL Bot"
    }
  ]
}
```

**SMTP Node (Outlook):**
```json
{
  "from": "rpa-fincheck@avito.ru",
  "to": "={{ $json.recipientEmails.join(';') }}",
  "subject": "={{ $json.emailSubject }}",
  "emailType": "html",
  "message": "={{ $json.emailBody }}",
  "attachments": "={{ [$input.first().binary.report] }}"
}
```

---

### Node 11: END

**Тип:** `n8n-nodes-base.noOp`

**Описание:** Завершение workflow

**Конфигурация:**
```json
{
  "notes": "Workflow completed successfully",
  "finalStatus": "={{ $json.status }}",
  "executionTime": "={{ Date.now() }}"
}
```

---

## 4. Обработка ошибок (Error Handling)

### 4.1 Error Trigger Node

**Тип:** `n8n-nodes-base.errorTrigger`

**Конфигурация:**
```json
{
  "workflowId": "FinCheck3PL_Trigger",
  "errorMessage": "={{ $json.error }}",
  "errorCode": "={{ $json.errorCode }}"
}
```

### 4.2 Стратегия Retry

Для критичных узлов (БД, API) реализуется логика повторных попыток:

**Code Node - Retry Logic:**
```javascript
const maxRetries = 3;
const currentRetry = $json.retryCount || 0;

if (currentRetry < maxRetries) {
  // Экспоненциальная задержка
  const delay = Math.pow(2, currentRetry) * 1000;
  
  return {
    json: {
      ...$json,
      retryCount: currentRetry + 1,
      nextRetryAt: new Date(Date.now() + delay).toISOString(),
      shouldRetry: true
    }
  };
} else {
  // Максимальное количество попыток исчерпано
  throw new Error(`SE-003: Max retries exceeded. Last error: ${$json.errorMessage}`);
}
```

### 4.3 Коды ошибок

| Код | Тип | Описание | Обработка |
|-----|-----|----------|-----------|
| SE-001 | System | Ошибка загрузки конфига | Retry 3x → Alert |
| SE-002 | System | Ошибка чтения файла | Retry 3x → Alert |
| SE-003 | System | Исчерпаны retry | Alert в MM |
| BE-001 | Business | Нет файлов в запросе | Уведомление инициатору |
| BE-002 | Business | Конфигурация СД не найдена | Уведомление инициатору |
| BE-003 | Business | Отсутствуют обязательные листы | Отчёт с ошибками |
| BE-004 | Business | Отсутствуют обязательные колонки | Отчёт с ошибками |
| BE-005 | Business | Лист пуст | Warning в отчёте |
| BE2-001 | Business | Нет данных для обработки | Завершение Success |

---

## 5. Конфигурационный файл (fincheck_config.json)

```json
{
  "deliveryServices": {
    "5post": {
      "displayName": "5post",
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
      "requiredSheets": [],
      "weightLimit": 15,
      "dateColumn": "Дата и время получения заявки",
      "statusCheck": false
    },
    "yandexCity": {
      "displayName": "Яндекс Внутригород",
      "requiredColumns": [
        "Статус заказа",
        "Стоимость заказа без НДС",
        "Дата заказа",
        "ID заявки",
        "Вес заказов",
        "Пройденное расстояние , км",
        "Cтоимость заказа без НДС, (AZ + BA + BG − BB) руб"
      ],
      "requiredSheets": [],
      "statusRule": {
        "column": "Статус заказа",
        "value": "Отмена",
        "costCondition": ">0"
      },
      "dateColumn": "Дата заказа",
      "statusCheck": true
    },
    "yandexPVZ": {
      "displayName": "Яндекс ПВЗ",
      "requiredColumns": [
        "Статус",
        "Стоимость без НДС",
        "Дата создания заказа",
        "ID заказа",
        "Вес"
      ],
      "statusRule": {
        "column": "Статус",
        "value": "отменен",
        "costCondition": ">0"
      },
      "statusCheck": true
    },
    "dostavista": {
      "displayName": "Достависта",
      "requiredColumns": [
        "Статус заказа",
        "Сумма заказа",
        "Дата заказа",
        "Номер заказа",
        "Вес"
      ],
      "statusRule": {
        "column": "Статус заказа",
        "value": "Отменен",
        "costCondition": ">0"
      },
      "statusCheck": true
    },
    "pochtaRossii": {
      "displayName": "Почта России",
      "requiredColumns": [
        "Дата приема",
        "Вес",
        "Стоимость",
        "Трекинг-номер"
      ],
      "weightLimit": 20,
      "statusCheck": false
    },
    "cdek": {
      "displayName": "СДЭК",
      "requiredColumns": [
        "Дата выдачи",
        "Вес",
        "Стоимость доставки",
        "Номер накладной"
      ],
      "weightLimit": 30,
      "statusCheck": false
    },
    "boxberry": {
      "displayName": "Boxberry",
      "requiredColumns": [
        "Дата отгрузки",
        "Вес",
        "Стоимость",
        "Номер заказа"
      ],
      "weightLimit": 25,
      "statusCheck": false
    },
    "dpd": {
      "displayName": "DPD",
      "requiredColumns": [
        "Дата доставки",
        "Вес места",
        "Стоимость",
        "Номер отправления"
      ],
      "weightLimit": 30,
      "statusCheck": false
    },
    "kurier": {
      "displayName": "Курьерская служба",
      "requiredColumns": [
        "Дата доставки",
        "Вес",
        "Стоимость",
        "Номер заказа"
      ],
      "statusCheck": false
    },
    "ems": {
      "displayName": "EMS",
      "requiredColumns": [
        "Дата приема",
        "Вес",
        "Стоимость",
        "Трекинг-номер"
      ],
      "weightLimit": 20,
      "statusCheck": false
    },
    "other": {
      "displayName": "Другая СД",
      "requiredColumns": [],
      "statusCheck": false
    }
  },
  
  "emailMapping": {
    "5post": {
      "cdEmails": ["support@5post.ru"],
      "avitoEmails": ["logistics@avito.ru"]
    },
    "yandexCity": {
      "cdEmails": ["support@yandex.ru"],
      "avitoEmails": ["logistics@avito.ru"]
    },
    "yandexPVZ": {
      "cdEmails": ["support@yandex.ru"],
      "avitoEmails": ["logistics@avito.ru"]
    },
    "dostavista": {
      "cdEmails": ["support@dostavista.ru"],
      "avitoEmails": ["logistics@avito.ru"]
    },
    "pochtaRossii": {
      "cdEmails": ["Svetlana.Kostenko@russianpost.ru"],
      "avitoEmails": ["benazaryan@avito.ru", "megolubeva@avito.ru"]
    },
    "cdek": {
      "cdEmails": ["support@cdek.ru"],
      "avitoEmails": ["logistics@avito.ru"]
    },
    "boxberry": {
      "cdEmails": ["support@boxberry.ru"],
      "avitoEmails": ["logistics@avito.ru"]
    },
    "dpd": {
      "cdEmails": ["support@dpd.ru"],
      "avitoEmails": ["logistics@avito.ru"]
    },
    "kurier": {
      "cdEmails": ["support@kurier.ru"],
      "avitoEmails": ["logistics@avito.ru"]
    },
    "ems": {
      "cdEmails": ["support@ems.ru"],
      "avitoEmails": ["logistics@avito.ru"]
    },
    "other": {
      "cdEmails": [],
      "avitoEmails": ["rpa-support@avito.ru"]
    }
  },
  
  "storage": {
    "inputPath": "/data/files/input/",
    "outputPath": "/data/files/output/",
    "tempPath": "/data/files/temp/",
    "archivePath": "/data/files/archive/",
    "dbConnection": "postgresql://n8n_user:n8n_pass@postgres:5432/rpa_db"
  },
  
  "integrations": {
    "mattermost": {
      "webhookUrl": "https://mattermost.avito.ru/hooks/xxx",
      "apiUrl": "https://mattermost.avito.ru/api/v4",
      "botUsername": "FinCheck3PL Bot"
    },
    "jupyter": {
      "baseUrl": "https://jupyter.avito.ru",
      "notebookPath": "/notebooks/fincheck_validation.ipynb",
      "timeout": 300000
    },
    "outlook": {
      "smtpHost": "smtp.office365.com",
      "smtpPort": 587,
      "secure": false,
      "requireTls": true,
      "fromAddress": "rpa-fincheck@avito.ru"
    }
  },
  
  "retryPolicy": {
    "maxRetries": 3,
    "initialDelay": 1000,
    "multiplier": 2,
    "maxDelay": 10000
  },
  
  "logging": {
    "level": "info",
    "destination": "file",
    "path": "/data/logs/executions/"
  }
}
```

---

## 6. Экспорт workflow для импорта в n8n

Workflow может быть экспортирован в формате JSON для импорта в n8n через UI или API.

**Структура export.json:**
```json
{
  "name": "FinCheck3PL_Trigger",
  "nodes": [
    {
      "parameters": { ... },
      "name": "Webhook Trigger",
      "type": "n8n-nodes-base.webhook",
      "position": [250, 300]
    },
    {
      "parameters": { ... },
      "name": "Init Config",
      "type": "n8n-nodes-base.code",
      "position": [450, 300]
    },
    ...
  ],
  "connections": {
    "Webhook Trigger": {
      "main": [
        [
          {
            "node": "Init Config",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    ...
  },
  "settings": {
    "executionOrder": "v1"
  }
}
```

---

## 7. Тестирование Workflow 1

### 7.1 Unit-тесты

| № | Тест | Входные данные | Ожидаемый результат |
|---|------|----------------|---------------------|
| 1 | Успешная инициализация | Валидный config.json | Status: initialized |
| 2 | Ошибка загрузки конфига | Отсутствует config.json | SE-001 error |
| 3 | Получение файлов | 1 Excel файл | Файл сохранён в input/ |
| 4 | Нет файлов | Пустой webhook | BE-001 error |
| 5 | Валидация колонок (успех) | Файл с полным набором колонок | isValid: true |
| 6 | Валидация колонок (ошибка) | Файл с недостающими колонками | isValid: false, BE-004 |
| 7 | Сохранение в БД | 100 записей | 100 записей в таблице |
| 8 | Нет данных в файле | Файл только с заголовками | BE2-001, completed_no_data |
| 9 | Вызов Workflow 2 | Валидные данные | Workflow 2 запущен |
| 10 | Отправка уведомлений | Результаты проверки | MM + Email отправлены |

### 7.2 Интеграционные тесты

**Сценарий 1: Полный цикл (успех)**
1. Отправить webhook с файлом 5post
2. Проверить сохранение файла
3. Проверить валидацию колонок
4. Проверить запись в БД
5. Проверить запуск Workflow 2
6. Проверить генерацию отчёта
7. Проверить отправку уведомлений

**Сценарий 2: Ошибка валидации**
1. Отправить webhook с некорректным файлом
2. Проверить обнаружение ошибок
3. Проверить отправку BE-уведомления
4. Проверить отсутствие записи в БД

**Сценарий 3: Нет данных**
1. Отправить webhook с пустым файлом
2. Проверить статус BE2-001
3. Проверить уведомление инициатору

---

## 8. Мониторинг и логирование

### 8.1 Логи выполнения

Каждый узел должен логировать:
```json
{
  "timestamp": "2026-02-16T10:15:25Z",
  "workflow": "FinCheck3PL_Trigger",
  "executionId": "exec_12345",
  "requestId": "req_67890",
  "node": "Validate Columns",
  "status": "success",
  "duration": 1250,
  "details": {
    "filesProcessed": 2,
    "errorsFound": 0
  }
}
```

### 8.2 Метрики для Grafana

- Количество выполнений workflow за период
- Среднее время выполнения
- Количество ошибок по типам (SE, BE, BE2)
- Количество обработанных файлов
- Количество записей в БД

---

## 9. Развёртывание

### 9.1 Переменные окружения

```bash
# Database
DB_TYPE=postgresdb
DB_POSTGRESDB_HOST=postgres
DB_POSTGRESDB_PORT=5432
DB_POSTGRESDB_DATABASE=rpa_db
DB_POSTGRESDB_USER=n8n_user
DB_POSTGRESDB_PASSWORD=n8n_pass

# Integrations
MATTERMOST_WEBHOOK_URL=https://mattermost.avito.ru/hooks/xxx
JUPYTER_BASE_URL=https://jupyter.avito.ru
OUTLOOK_SMTP_HOST=smtp.office365.com
OUTLOOK_SMTP_PORT=587

# Storage
FILES_INPUT_PATH=/data/files/input/
FILES_OUTPUT_PATH=/data/files/output/
CONFIG_PATH=/data/config/
```

### 9.2 Docker Compose

См. раздел 9.1 в файле `FinCheck3PL_n8n_Analysis.md`.

---

## 10. Приложение: Чек-лист развёртывания Workflow 1

- [ ] Создать директорию `/data/config/` и разместить `fincheck_config.json`
- [ ] Создать директории для файлов (`input`, `output`, `temp`, `archive`)
- [ ] Настроить PostgreSQL базу данных и таблицу `fincheck_orders`
- [ ] Создать Webhook в Mattermost и получить URL
- [ ] Настроить SMTP подключение к Outlook
- [ ] Проверить доступность Jupyter API
- [ ] Импортировать workflow в n8n
- [ ] Протестировать каждый узел отдельно
- [ ] Провести полный интеграционный тест
- [ ] Настроить мониторинг в Grafana
- [ ] Документировать процесс для команды поддержки

---

*Документ подготовлен для разработки Workflow 1: FinCheck3PL_Trigger*

*Версия: 1.0*  
*Дата: 2026-02-16*
