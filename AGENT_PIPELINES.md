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

## 2. Пайплайн графовой памяти (Graph Memory Pipeline)

**Назначение:** Построение и обновление графа знаний из разговора путем извлечения сущностей и их отношений.

**Используемые промпты:**
1. Entity Extraction Prompt (inline)
2. EXTRACT_RELATIONS_PROMPT
3. UPDATE_GRAPH_PROMPT
4. DELETE_RELATIONS_SYSTEM_PROMPT

**Входные данные:** Текстовые данные от пользователя

**Выходные данные:** Обновленный граф знаний с новыми/измененными/удаленными отношениями

### Схема пайплайна

```
┌──────────────────────────────────────────────────────────────────────┐
│                       GRAPH MEMORY PIPELINE                          │
└──────────────────────────────────────────────────────────────────────┘

INPUT: data = "Alice works as a professor at Stanford University."
       filters = {"user_id": "alice_123"}

        │
        ▼
┌───────────────────────────────────────────────────────────────────────┐
│  ШАГ 1: Извлечение сущностей                                          │
│  Prompt: Entity Extraction Prompt (inline)                            │
│  Tool: EXTRACT_ENTITIES_TOOL / EXTRACT_ENTITIES_STRUCT_TOOL          │
│                                                                       │
│  LLM Input:                                                           │
│    - System: "You are a smart assistant who understands entities..."  │
│              (with user_id replaced: "alice_123")                     │
│    - User: "{data}"                                                   │
│    - Tools: [EXTRACT_ENTITIES_TOOL]                                  │
│                                                                       │
│  LLM Output: Function Call                                            │
│    {                                                                  │
│      "function": "extract_entities",                                 │
│      "arguments": {                                                   │
│        "entities": [                                                  │
│          {"entity": "alice_123", "entity_type": "person"},           │
│          {"entity": "Stanford University", "entity_type": "org"}     │
│        ]                                                              │
│      }                                                                │
│    }                                                                  │
└───────────────────────────────────────────────────────────────────────┘
        │
        │ entity_type_map = {
        │   "alice_123": "person",
        │   "Stanford University": "org"
        │ }
        ▼
┌───────────────────────────────────────────────────────────────────────┐
│  ШАГ 2: Установление отношений между сущностями                      │
│  Prompt: EXTRACT_RELATIONS_PROMPT                                     │
│  Tool: RELATIONS_TOOL / RELATIONS_STRUCT_TOOL                        │
│                                                                       │
│  LLM Input:                                                           │
│    - System: EXTRACT_RELATIONS_PROMPT (with CUSTOM_PROMPT filled)    │
│    - User: "Entities: {entities}\n\nText: {data}"                    │
│    - Tools: [RELATIONS_TOOL]                                         │
│                                                                       │
│  LLM Output: Function Call                                            │
│    {                                                                  │
│      "function": "establish_relationships",                          │
│      "arguments": {                                                   │
│        "entities": [                                                  │
│          {                                                            │
│            "source": "alice_123",                                    │
│            "relationship": "works_as_professor_at",                  │
│            "destination": "Stanford University"                      │
│          }                                                            │
│        ]                                                              │
│      }                                                                │
│    }                                                                  │
└───────────────────────────────────────────────────────────────────────┘
        │
        │ relations = [
        │   {
        │     "source": "alice_123",
        │     "relationship": "works_as_professor_at",
        │     "destination": "Stanford University",
        │     "source_type": "person",
        │     "destination_type": "org"
        │   }
        │ ]
        ▼
┌────────────────────────────────────────┐
│  ШАГ 3: Поиск существующих отношений  │
│  Database: Graph DB Search             │
│  Query: Relationships for entities     │
└────────────────────────────────────────┘
        │
        │ existing_relations = [
        │   "alice_123 -- works_at -- MIT",
        │   "alice_123 -- lives_in -- Boston"
        │ ]
        ▼
┌───────────────────────────────────────────────────────────────────────┐
│  ШАГ 4A: Определение отношений для удаления                          │
│  Prompt: DELETE_RELATIONS_SYSTEM_PROMPT                               │
│  Tool: DELETE_MEMORY_TOOL_GRAPH / DELETE_MEMORY_STRUCT_TOOL_GRAPH    │
│                                                                       │
│  LLM Input:                                                           │
│    - System: DELETE_RELATIONS_SYSTEM_PROMPT                           │
│    - User: "Existing: {existing_relations}\n\nNew: {data}"           │
│    - Tools: [DELETE_MEMORY_TOOL_GRAPH]                              │
│                                                                       │
│  LLM Output: Function Call (если нужно удалить)                       │
│    {                                                                  │
│      "function": "delete_graph_memory",                              │
│      "arguments": {                                                   │
│        "source": "alice_123",                                        │
│        "relationship": "works_at",                                   │
│        "destination": "MIT"                                          │
│      }                                                                │
│    }                                                                  │
│                                                                       │
│  Reasoning: Alice now works at Stanford, so old MIT relation         │
│              is outdated and should be deleted                        │
└───────────────────────────────────────────────────────────────────────┘
        │
        │ to_delete = [
        │   {"source": "alice_123", "relationship": "works_at",
        │    "destination": "MIT"}
        │ ]
        ▼
┌───────────────────────────────────────────────────────────────────────┐
│  ШАГ 4B: Определение отношений для обновления                        │
│  Prompt: UPDATE_GRAPH_PROMPT                                          │
│  Tool: UPDATE_MEMORY_TOOL_GRAPH / UPDATE_MEMORY_STRUCT_TOOL_GRAPH    │
│                                                                       │
│  LLM Input:                                                           │
│    - System: UPDATE_GRAPH_PROMPT.format(                             │
│                existing_memories=existing_relations,                  │
│                new_memories=new_relations)                            │
│    - Tools: [UPDATE_MEMORY_TOOL_GRAPH]                              │
│                                                                       │
│  LLM Output: Function Call (если нужно обновить)                      │
│    {                                                                  │
│      "function": "update_graph_memory",                              │
│      "arguments": {                                                   │
│        "source": "alice_123",                                        │
│        "destination": "Stanford University",                         │
│        "relationship": "professor"  # Refined relationship           │
│      }                                                                │
│    }                                                                  │
└───────────────────────────────────────────────────────────────────────┘
        │
        │ to_update = [...]
        ▼
┌────────────────────────────────────────────────────────┐
│  ШАГ 5: Применение изменений к Graph DB               │
│  - DELETE устаревших отношений                         │
│  - ADD новых сущностей и отношений                     │
│  - UPDATE существующих отношений                       │
│  - Cypher queries to Neo4j/Memgraph/Kuzu/Neptune      │
└────────────────────────────────────────────────────────┘
        │
        ▼
OUTPUT: {
  "deleted_entities": [
    "alice_123 -- works_at -- MIT"
  ],
  "added_entities": [
    "alice_123 -- works_as_professor_at -- Stanford University"
  ]
}
```

### Детали передачи данных

**INPUT → ШАГ 1:**
- **Входные данные**: Текст сообщения + user_id в filters
- **Обработка**: user_id заменяет само-ссылки в промпте

**ШАГ 1 → ШАГ 2:**
- **Выходные данные**: Список сущностей с типами через function call
- **Формат**: `[{"entity": "name", "entity_type": "type"}, ...]`
- **Создается**: entity_type_map для последующего использования

**ШАГ 2 → ШАГ 3:**
- **Выходные данные**: Список отношений (триплеты) через function call
- **Формат**: `[{"source": "A", "relationship": "R", "destination": "B"}, ...]`
- **Обогащается**: Добавляются типы сущностей из entity_type_map

**ШАГ 3 → ШАГ 4:**
- **Поиск**: Graph DB query для получения существующих отношений
- **Формат существующих**: `["source -- relationship -- destination", ...]`
- **Используется**: Для определения DELETE и UPDATE операций

**ШАГ 4 → ШАГ 5:**
- **DELETE**: Список отношений для удаления (противоречия, устаревшие)
- **UPDATE**: Список отношений для обновления (уточнение, дополнение)
- **ADD**: Новые отношения (из ШАГ 2, минус DELETE/UPDATE)

### Особенности Graph Pipeline

1. **Обработка само-ссылок**: "I", "me", "my" → user_id из filters
2. **Два прохода LLM**: Сначала entities, потом relations (нельзя параллельно)
3. **Два вида обновлений**: DELETE (противоречия) и UPDATE (уточнение)
4. **Санитизация**: Отношения санитизируются для Cypher queries
5. **Поддержка разных Graph DB**: Neo4j, Memgraph, Kuzu, Neptune

---

