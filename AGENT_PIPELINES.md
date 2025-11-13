# Документация по пайплайнам агента Mem0

Этот документ описывает все возможные пайплайны работы агента Mem0, последовательность вызова промптов, передачу данных между этапами и схемы работы.

---

## 1. Базовый пайплайн создания памяти (Basic Memory Creation Pipeline)

**Назначение:** Извлечение фактов из разговора и сохранение их в памяти системы.

**Используемые промпты:**
1. USER_MEMORY_EXTRACTION_PROMPT или AGENT_MEMORY_EXTRACTION_PROMPT
2. DEFAULT_UPDATE_MEMORY_PROMPT

**Входные данные:** Массив сообщений (messages) от пользователя и ассистента

**Выходные данные:** Обновленная память с новыми/измененными/удаленными фактами

### Схема пайплайна

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         BASIC MEMORY PIPELINE                           │
└─────────────────────────────────────────────────────────────────────────┘

INPUT: messages = [
  {"role": "user", "content": "Hi, my name is Alice. I love pizza."},
  {"role": "assistant", "content": "Nice to meet you, Alice!"}
]

        │
        ▼
┌─────────────────────────────────┐
│  ШАГ 1: Парсинг сообщений       │
│  Function: parse_messages()     │
└─────────────────────────────────┘
        │
        │ Parsed messages text
        ▼
┌───────────────────────────────────────────────────────────────────┐
│  ШАГ 2: Извлечение фактов                                         │
│  Prompt: USER_MEMORY_EXTRACTION_PROMPT (for user facts) OR       │
│          AGENT_MEMORY_EXTRACTION_PROMPT (for agent facts)         │
│                                                                   │
│  LLM Input:                                                       │
│    - System: USER_MEMORY_EXTRACTION_PROMPT                        │
│    - User: "Input:\n{parsed_messages}"                           │
│                                                                   │
│  LLM Output Format: JSON                                          │
│    {"facts": ["Name is Alice", "Loves pizza"]}                   │
└───────────────────────────────────────────────────────────────────┘
        │
        │ extracted_facts = ["Name is Alice", "Loves pizza"]
        ▼
┌────────────────────────────────────┐
│  ШАГ 3: Получение текущей памяти  │
│  Database: Vector Store Search     │
└────────────────────────────────────┘
        │
        │ existing_memory = [
        │   {"id": "0", "text": "Name is Bob"},
        │   {"id": "1", "text": "Likes burgers"}
        │ ]
        ▼
┌──────────────────────────────────────────────────────────────────┐
│  ШАГ 4: Определение операций обновления памяти                  │
│  Prompt: DEFAULT_UPDATE_MEMORY_PROMPT                            │
│  Function: get_update_memory_messages()                          │
│                                                                  │
│  LLM Input:                                                      │
│    - System: DEFAULT_UPDATE_MEMORY_PROMPT                        │
│    - User: Current memory + Retrieved facts                     │
│                                                                  │
│  LLM Output Format: JSON with operations                         │
│  {                                                               │
│    "memory": [                                                   │
│      {"id": "0", "text": "Name is Alice",                       │
│       "event": "UPDATE", "old_memory": "Name is Bob"},          │
│      {"id": "1", "text": "Likes burgers", "event": "NONE"},    │
│      {"id": "2", "text": "Loves pizza", "event": "ADD"}        │
│    ]                                                             │
│  }                                                               │
└──────────────────────────────────────────────────────────────────┘
        │
        │ memory_operations = [UPDATE id:0, NONE id:1, ADD id:2]
        ▼
┌─────────────────────────────────────────┐
│  ШАГ 5: Применение операций к БД        │
│  - DELETE старых записей (UPDATE/DELETE)│
│  - ADD новых записей (UPDATE/ADD)       │
│  - Генерация embeddings                 │
│  - Сохранение в Vector Store            │
└─────────────────────────────────────────┘
        │
        ▼
OUTPUT: {
  "results": [
    {"event": "update", "data": "Name is Alice", "id": "mem_xxx"},
    {"event": "add", "data": "Loves pizza", "id": "mem_yyy"}
  ]
}
```

### Детали передачи данных

**ШАГ 1 → ШАГ 2:**
- **Входные данные**: Массив messages
- **Выходные данные**: Форматированный текст разговора
- **Формат**: "user: {content}\nassistant: {content}\n..."

**ШАГ 2 → ШАГ 3:**
- **Входные данные**: JSON с полем "facts" (массив строк)
- **Выходные данные**: Список извлеченных фактов
- **Формат**: `["Fact 1", "Fact 2", ...]`

**ШАГ 3 → ШАГ 4:**
- **Входные данные**: Запрос к Vector Store на основе новых фактов
- **Выходные данные**: Существующие релевантные записи памяти
- **Формат**: `[{"id": "X", "text": "Memory text", "score": 0.9}, ...]`

**ШАГ 4 → ШАГ 5:**
- **Входные данные**: Существующая память + новые факты
- **Выходные данные**: JSON с операциями (ADD/UPDATE/DELETE/NONE)
- **Формат**: `{"memory": [{"id": "X", "text": "...", "event": "ADD/UPDATE/DELETE/NONE", "old_memory": "..."}]}`

**ШАГ 5 → OUTPUT:**
- **Операции**: Применение изменений к базе данных
- **Результат**: Список выполненных операций с ID новых записей

---

