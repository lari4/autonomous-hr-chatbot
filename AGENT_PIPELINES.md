# Схемы Работы HR Agent (Agent Pipelines)
## Autonomous HR Chatbot

Данный документ описывает все возможные пайплайны (схемы работы) HR агента, последовательность вызова промптов, передачу данных между компонентами и различные сценарии обработки запросов пользователя.

---

## Оглавление

1. [Общая Архитектура ReAct Agent](#1-общая-архитектура-react-agent)
2. [Пайплайн 1: Запрос к Политикам (Timekeeping Policies)](#2-пайплайн-1-запрос-к-политикам)
3. [Пайплайн 2: Запрос к Данным Сотрудника (Employee Data)](#3-пайплайн-2-запрос-к-данным-сотрудника)
4. [Пайплайн 3: Математические Вычисления (Calculator)](#4-пайплайн-3-математические-вычисления)
5. [Пайплайн 4: Комбинированный - Политики + Данные](#5-пайплайн-4-комбинированный---политики--данные)
6. [Пайплайн 5: Комбинированный - Данные + Калькулятор](#6-пайплайн-5-комбинированный---данные--калькулятор)
7. [Пайплайн 6: Комбинированный - Все Инструменты](#7-пайплайн-6-комбинированный---все-инструменты)
8. [Пайплайн 7: Прямой Ответ (Без Инструментов)](#8-пайплайн-7-прямой-ответ)

---

## 1. Общая Архитектура ReAct Agent

HR Chatbot использует архитектуру **Zero-Shot ReAct** (Reasoning + Acting), где агент циклически выполняет:

```
┌────────────────────────────────────────────────────────────────┐
│                    REACT AGENT CYCLE                           │
│                                                                │
│  Шаг 1: THOUGHT (Размышление)                                 │
│          ↓ Агент анализирует вопрос                           │
│  Шаг 2: ACTION (Действие)                                     │
│          ↓ Агент выбирает инструмент                          │
│  Шаг 3: ACTION INPUT (Ввод для инструмента)                  │
│          ↓ Агент формирует запрос к инструменту               │
│  Шаг 4: OBSERVATION (Наблюдение)                             │
│          ↓ Агент получает результат от инструмента            │
│                                                                │
│  → Если ответ получен: FINAL ANSWER                           │
│  → Если нужно больше данных: возврат к Шагу 1                │
└────────────────────────────────────────────────────────────────┘
```

**Доступные инструменты:**
1. **Timekeeping Policies** - векторный поиск по политикам (Pinecone + RetrievalQA)
2. **Employee Data** - выполнение Pandas операций на DataFrame
3. **Calculator** - математические вычисления через LLMMathChain

---

## 2. Пайплайн 1: Запрос к Политикам

**Сценарий:** Пользователь спрашивает о политиках компании, правилах отпусков, больничных и т.д.

**Пример запроса:** *"What is the policy on unused vacation leave?"*

### Схема потока данных:

```
┌──────────────────────────────────────────────────────────────────────────┐
│ USER INPUT                                                               │
│ "What is the policy on unused vacation leave?"                          │
└────────────────────────────┬─────────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ STEP 1: AGENT INITIALIZATION                                             │
│                                                                          │
│ Системный промпт загружается:                                           │
│ "You are friendly HR assistant. You are tasked to assist               │
│  the current user: Alexander Verdad on questions related to HR.         │
│  You have access to the following tools:"                               │
│                                                                          │
│ + Описания всех 3 инструментов                                          │
│ + ReAct формат инструкций                                               │
└────────────────────────────┬─────────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ STEP 2: THOUGHT (LLM Processing)                                         │
│                                                                          │
│ INPUT:                                                                   │
│   - Системный промпт                                                     │
│   - Вопрос пользователя                                                  │
│   - Описания инструментов                                                │
│                                                                          │
│ LLM REASONING:                                                           │
│ "I need to check the timekeeping policies to answer this question       │
│  about unused vacation leave policy."                                    │
│                                                                          │
│ OUTPUT:                                                                  │
│   Thought: "I need to check timekeeping policies"                       │
└────────────────────────────┬─────────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ STEP 3: ACTION SELECTION                                                 │
│                                                                          │
│ LLM OUTPUT:                                                              │
│   Action: Timekeeping Policies                                          │
│   Action Input: "Vacation Leave Policy - Unused Leave"                  │
└────────────────────────────┬─────────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ STEP 4: TOOL EXECUTION - Timekeeping Policies                           │
│                                                                          │
│ 4.1 EMBEDDING GENERATION                                                 │
│     ┌────────────────────────────────────────┐                          │
│     │ OpenAI Embeddings API                  │                          │
│     │ Model: text-embedding-ada-002          │                          │
│     │                                         │                          │
│     │ INPUT: "Vacation Leave Policy -        │                          │
│     │         Unused Leave"                   │                          │
│     │                                         │                          │
│     │ OUTPUT: [0.123, -0.456, 0.789, ...]   │                          │
│     │         (1536-dimensional vector)       │                          │
│     └──────────────┬─────────────────────────┘                          │
│                    │                                                     │
│                    ▼                                                     │
│ 4.2 VECTOR SEARCH IN PINECONE                                           │
│     ┌────────────────────────────────────────┐                          │
│     │ Pinecone Vector Database               │                          │
│     │ Index: tk-policy                       │                          │
│     │                                         │                          │
│     │ OPERATION: Similarity Search           │                          │
│     │   - Query Vector: от шага 4.1          │                          │
│     │   - Top K: 4 (по умолчанию)           │                          │
│     │                                         │                          │
│     │ RETURNS: 4 наиболее релевантных       │                          │
│     │          фрагмента из hr_policy.txt    │                          │
│     └──────────────┬─────────────────────────┘                          │
│                    │                                                     │
│                    ▼                                                     │
│ 4.3 RETRIEVAL QA CHAIN                                                  │
│     ┌────────────────────────────────────────┐                          │
│     │ RetrievalQA Chain                      │                          │
│     │ Chain Type: "stuff"                    │                          │
│     │                                         │                          │
│     │ PROMPT CONSTRUCTION:                   │                          │
│     │ ┌────────────────────────────────────┐ │                          │
│     │ │ Use the following pieces of        │ │                          │
│     │ │ context to answer the question.    │ │                          │
│     │ │                                     │ │                          │
│     │ │ Context:                            │ │                          │
│     │ │ [Chunk 1]: "A. Vacation Leave...   │ │                          │
│     │ │  Unused Leave can be carried over  │ │                          │
│     │ │  to the next year. However, the    │ │                          │
│     │ │  total accumulated leave should    │ │                          │
│     │ │  not exceed 30 days. Any excess    │ │                          │
│     │ │  leave will be forfeited."         │ │                          │
│     │ │                                     │ │                          │
│     │ │ [Chunk 2]: "Encashment: Unused     │ │                          │
│     │ │  Vacation Leave can be encashed    │ │                          │
│     │ │  at the end of the year at the     │ │                          │
│     │ │  basic salary rate..."             │ │                          │
│     │ │                                     │ │                          │
│     │ │ [Chunk 3]: ...                      │ │                          │
│     │ │ [Chunk 4]: ...                      │ │                          │
│     │ │                                     │ │                          │
│     │ │ Question: Vacation Leave Policy -  │ │                          │
│     │ │           Unused Leave             │ │                          │
│     │ └────────────────────────────────────┘ │                          │
│     │          ↓                              │                          │
│     │    LLM PROCESSING                      │                          │
│     │    (GPT-3.5-turbo)                     │                          │
│     │          ↓                              │                          │
│     │ OUTPUT: Synthesized answer             │                          │
│     └──────────────┬─────────────────────────┘                          │
│                    │                                                     │
│ TOOL OUTPUT:                                                            │
│ "According to the company policy, unused Vacation Leave can be         │
│  carried over to the next year. However, the total accumulated         │
│  leave should not exceed 30 days. Any excess leave will be             │
│  forfeited. Additionally, unused Vacation Leave can be encashed        │
│  at the end of the year at the basic salary rate, calculated as        │
│  basic salary divided by 30 days multiplied by unused leaves."         │
└────────────────────────────┬─────────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ STEP 5: OBSERVATION                                                      │
│                                                                          │
│ Агент получает результат от инструмента:                                │
│                                                                          │
│ Observation: "According to the company policy, unused Vacation          │
│ Leave can be carried over to the next year..."                          │
└────────────────────────────┬─────────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ STEP 6: FINAL THOUGHT                                                    │
│                                                                          │
│ LLM PROCESSING:                                                          │
│ INPUT: История всего диалога + Observation                              │
│                                                                          │
│ OUTPUT:                                                                  │
│ Thought: "I now know the final answer"                                  │
└────────────────────────────┬─────────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ STEP 7: FINAL ANSWER                                                     │
│                                                                          │
│ Final Answer: "According to the company policy, unused Vacation         │
│ Leave can be carried over to the next year, but the total accumulated  │
│ leave should not exceed 30 days. Any excess leave will be forfeited.   │
│ Additionally, you can encash unused Vacation Leave at the end of the    │
│ year at your basic salary rate (basic salary ÷ 30 × unused days)."     │
└────────────────────────────┬─────────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ USER OUTPUT                                                              │
│ [Ответ отображается пользователю через Gradio UI]                      │
└──────────────────────────────────────────────────────────────────────────┘
```

### Детали передачи данных:

**Промпты и их последовательность:**

1. **Системный промпт** → LLM (Thought генерация)
2. **Action Input промпт** → OpenAI Embeddings API
3. **Embedding vector** → Pinecone Vector DB
4. **Retrieved chunks** → RetrievalQA Chain промпт
5. **QA промпт с контекстом** → LLM (Answer генерация)
6. **Tool output** → Agent (Observation)
7. **Observation** → LLM (Final Answer генерация)

**Вовлеченные компоненты:**
- `ChatOpenAI` (gpt-3.5-turbo) - основной LLM
- `OpenAIEmbeddings` (text-embedding-ada-002) - генерация эмбеддингов
- `Pinecone` - векторная база данных
- `RetrievalQA` - chain для обработки QA с контекстом
- `Tool` wrapper - обертка для инструмента

**Количество вызовов LLM:** 3
1. Thought + Action selection
2. QA answer generation (внутри RetrievalQA)
3. Final Answer generation

---

## 3. Пайплайн 2: Запрос к Данным Сотрудника

**Сценарий:** Пользователь спрашивает о своих личных данных, балансе отпусков, зарплате, супервайзере и т.д.

**Пример запроса:** *"How many sick leaves do I have left?"*

### Схема потока данных:

```
┌──────────────────────────────────────────────────────────────────────────┐
│ USER INPUT                                                               │
│ "How many sick leaves do I have left?"                                  │
└────────────────────────────┬─────────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ STEP 1: AGENT RECEIVES INPUT                                             │
│                                                                          │
│ Системный промпт + вопрос пользователя                                  │
│ User context: Alexander Verdad                                          │
└────────────────────────────┬─────────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ STEP 2: THOUGHT (LLM Processing)                                         │
│                                                                          │
│ INPUT:                                                                   │
│   - Системный промпт                                                     │
│   - Вопрос: "How many sick leaves do I have left?"                      │
│   - Employee Data tool description с примером:                          │
│     "<user>: How many Sick Leave do I have left?                        │
│      <assistant>: df[df['name'] == 'Alexander Verdad']['sick_leave']"  │
│   - Информация о колонках DataFrame                                      │
│                                                                          │
│ LLM REASONING:                                                           │
│ "I need to check the employee data to find out how many sick leaves    │
│  the user has left. I'll use the Employee Data tool with pandas."       │
│                                                                          │
│ OUTPUT:                                                                  │
│   Thought: "I need to query the employee dataframe"                     │
└────────────────────────────┬─────────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ STEP 3: ACTION SELECTION                                                 │
│                                                                          │
│ LLM OUTPUT:                                                              │
│   Action: Employee Data                                                  │
│   Action Input: "df[df['name'] == 'Alexander Verdad']['sick_leave']"   │
└────────────────────────────┬─────────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ STEP 4: TOOL EXECUTION - Employee Data (PythonAstREPLTool)              │
│                                                                          │
│ 4.1 TOOL INITIALIZATION (уже выполнено при запуске)                     │
│     ┌────────────────────────────────────────┐                          │
│     │ DataFrame Loading                      │                          │
│     │                                         │                          │
│     │ SOURCE: employee_data.csv              │                          │
│     │                                         │                          │
│     │ df = pd.read_csv("employee_data.csv")  │                          │
│     │                                         │                          │
│     │ COLUMNS:                                │                          │
│     │  - employee_id                         │                          │
│     │  - name                                 │                          │
│     │  - position                             │                          │
│     │  - organizational_unit                  │                          │
│     │  - rank                                 │                          │
│     │  - hire_date                            │                          │
│     │  - regularization_date                  │                          │
│     │  - vacation_leave                       │                          │
│     │  - sick_leave          ← TARGET        │                          │
│     │  - basic_pay_in_php                    │                          │
│     │  - employment_status                    │                          │
│     │  - supervisor                           │                          │
│     └────────────────────────────────────────┘                          │
│                                                                          │
│ 4.2 PYTHON CODE EXECUTION                                               │
│     ┌────────────────────────────────────────┐                          │
│     │ PythonAstREPLTool                      │                          │
│     │ Safe Python AST Execution              │                          │
│     │                                         │                          │
│     │ INPUT CODE (from Action Input):        │                          │
│     │ ┌────────────────────────────────────┐ │                          │
│     │ │ df[df['name'] == 'Alexander        │ │                          │
│     │ │    Verdad']['sick_leave']          │ │                          │
│     │ └────────────────────────────────────┘ │                          │
│     │          ↓                              │                          │
│     │ EXECUTION STEPS:                       │                          │
│     │                                         │                          │
│     │ 1. Filter DataFrame:                   │                          │
│     │    df['name'] == 'Alexander Verdad'   │                          │
│     │    → Boolean mask                      │                          │
│     │                                         │                          │
│     │ 2. Apply filter:                       │                          │
│     │    df[mask]                            │                          │
│     │    → Filtered DataFrame (1 row)       │                          │
│     │                                         │                          │
│     │ 3. Select column:                      │                          │
│     │    filtered_df['sick_leave']          │                          │
│     │    → Pandas Series                     │                          │
│     │                                         │                          │
│     │ RESULT:                                 │                          │
│     │ ┌────────────────────────────────────┐ │                          │
│     │ │ 0    12                            │ │                          │
│     │ │ Name: sick_leave, dtype: int64     │ │                          │
│     │ └────────────────────────────────────┘ │                          │
│     │          ↓                              │                          │
│     │ TOOL OUTPUT: "0    12                  │                          │
│     │ Name: sick_leave, dtype: int64"        │                          │
│     └────────────────────────────────────────┘                          │
│                                                                          │
│ NOTE: PythonAstREPLTool использует AST (Abstract Syntax Tree)          │
│       для безопасного выполнения кода - предотвращает выполнение       │
│       опасных операций (exec, eval, import и т.д.)                      │
└────────────────────────────┬─────────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ STEP 5: OBSERVATION                                                      │
│                                                                          │
│ Агент получает результат:                                               │
│                                                                          │
│ Observation: "0    12                                                   │
│ Name: sick_leave, dtype: int64"                                         │
│                                                                          │
│ Agent scratchpad обновляется:                                           │
│   Question: How many sick leaves do I have left?                        │
│   Thought: I need to query employee dataframe                           │
│   Action: Employee Data                                                  │
│   Action Input: df[df['name'] == 'Alexander Verdad']['sick_leave']     │
│   Observation: 0    12                                                  │
│               Name: sick_leave, dtype: int64                            │
└────────────────────────────┬─────────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ STEP 6: FINAL THOUGHT                                                    │
│                                                                          │
│ LLM PROCESSING:                                                          │
│ INPUT: Весь agent_scratchpad с историей рассуждений                     │
│                                                                          │
│ LLM REASONING:                                                           │
│ "Based on the observation, I can see that the sick_leave value is 12.  │
│  I now have enough information to provide a clear answer."              │
│                                                                          │
│ OUTPUT:                                                                  │
│ Thought: "I now know the final answer"                                  │
└────────────────────────────┬─────────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ STEP 7: FINAL ANSWER GENERATION                                          │
│                                                                          │
│ LLM формирует человекочитаемый ответ из технических данных:            │
│                                                                          │
│ INPUT: "0    12\nName: sick_leave, dtype: int64"                        │
│                                                                          │
│ OUTPUT (Final Answer):                                                   │
│ "You have 12 sick leaves left."                                         │
└────────────────────────────┬─────────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ USER OUTPUT                                                              │
│ "You have 12 sick leaves left."                                         │
└──────────────────────────────────────────────────────────────────────────┘
```

### Альтернативные Pandas операции:

Employee Data tool может выполнять различные Pandas операции:

**1. Простое извлечение данных:**
```python
Action Input: df[df['name'] == 'Alexander Verdad']['vacation_leave']
→ Возвращает количество дней отпуска
```

**2. Множественные колонки:**
```python
Action Input: df[df['name'] == 'Alexander Verdad'][['vacation_leave', 'sick_leave']]
→ Возвращает DataFrame с несколькими колонками
```

**3. Агрегация:**
```python
Action Input: df.groupby('organizational_unit')['basic_pay_in_php'].mean()
→ Средняя зарплата по отделам
```

**4. Фильтрация и подсчет:**
```python
Action Input: len(df[df['employment_status'] == 'Regular'])
→ Количество постоянных сотрудников
```

**5. Сравнение:**
```python
Action Input: df[df['basic_pay_in_php'] > 50000]['name'].tolist()
→ Список имен с зарплатой выше 50000
```

### Детали передачи данных:

**Промпты и их последовательность:**

1. **Системный промпт + Employee Data description** → LLM (Thought)
2. **Generated Pandas code** → PythonAstREPLTool
3. **Pandas execution result** → Agent (Observation)
4. **Observation** → LLM (Final Answer)

**Вовлеченные компоненты:**
- `ChatOpenAI` (gpt-3.5-turbo) - основной LLM
- `PythonAstREPLTool` - безопасный Python executor
- `pandas.DataFrame` - данные из employee_data.csv
- `Tool` wrapper - обертка для инструмента

**Количество вызовов LLM:** 2
1. Thought + Action selection + Code generation
2. Final Answer generation

**Безопасность:**
- `PythonAstREPLTool` использует AST парсинг
- Блокирует опасные операции: `exec`, `eval`, `import`, `__import__`
- Ограничивает доступ только к предопределенным переменным (`df`)
- Предотвращает файловые операции и системные вызовы

---

## 4. Пайплайн 3: Математические Вычисления

**Сценарий:** Пользователь задает вопрос, требующий математических расчетов.

**Пример запроса:** *"If I take 5 vacation days, how many will I have left?"*

### Схема потока данных:

```
┌──────────────────────────────────────────────────────────────────────────┐
│ USER INPUT                                                               │
│ "If I take 5 vacation days, how many will I have left?"                │
└────────────────────────────┬─────────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ STEP 1: AGENT RECEIVES INPUT                                             │
│                                                                          │
│ Системный промпт + вопрос пользователя                                  │
└────────────────────────────┬─────────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ STEP 2: THOUGHT #1 - Нужны данные                                       │
│                                                                          │
│ LLM REASONING:                                                           │
│ "To answer this, I first need to know how many vacation days the       │
│  user currently has."                                                    │
│                                                                          │
│ OUTPUT:                                                                  │
│   Thought: "I need to check current vacation leave balance"             │
│   Action: Employee Data                                                  │
│   Action Input: "df[df['name'] == 'Alexander Verdad']['vacation_leave']"│
└────────────────────────────┬─────────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ STEP 3: EMPLOYEE DATA TOOL EXECUTION                                     │
│                                                                          │
│ PythonAstREPLTool выполняет Pandas код                                  │
│                                                                          │
│ RESULT: "0    15                                                        │
│         Name: vacation_leave, dtype: int64"                             │
└────────────────────────────┬─────────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ STEP 4: OBSERVATION #1                                                   │
│                                                                          │
│ Observation: User has 15 vacation days                                  │
│                                                                          │
│ Agent scratchpad:                                                        │
│   Question: If I take 5 vacation days, how many will I have left?      │
│   Thought: I need to check current vacation leave balance               │
│   Action: Employee Data                                                  │
│   Action Input: df[df['name'] == 'Alexander Verdad']['vacation_leave'] │
│   Observation: 0    15                                                  │
│               Name: vacation_leave, dtype: int64                        │
└────────────────────────────┬─────────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ STEP 5: THOUGHT #2 - Нужен расчет                                       │
│                                                                          │
│ LLM REASONING:                                                           │
│ "Now I know the user has 15 vacation days. I need to calculate         │
│  15 - 5 to find out how many will be left."                            │
│                                                                          │
│ OUTPUT:                                                                  │
│   Thought: "I need to calculate 15 - 5"                                │
│   Action: Calculator                                                     │
│   Action Input: "15 - 5"                                                │
└────────────────────────────┬─────────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ STEP 6: TOOL EXECUTION - Calculator (LLMMathChain)                      │
│                                                                          │
│ 6.1 LLMMATHCHAIN PROMPT CONSTRUCTION                                    │
│     ┌────────────────────────────────────────┐                          │
│     │ LLMMathChain Internal Prompt           │                          │
│     │                                         │                          │
│     │ TEMPLATE:                               │                          │
│     │ ┌────────────────────────────────────┐ │                          │
│     │ │ Translate a math problem into a    │ │                          │
│     │ │ expression that can be executed    │ │                          │
│     │ │ using Python's numexpr library.    │ │                          │
│     │ │ Use the output of running this     │ │                          │
│     │ │ code to answer the question.       │ │                          │
│     │ │                                     │ │                          │
│     │ │ Question: ${Question with          │ │                          │
│     │ │            math problem}            │ │                          │
│     │ │                                     │ │                          │
│     │ │ ```text                             │ │                          │
│     │ │ ${single line mathematical          │ │                          │
│     │ │   expression that solves it}        │ │                          │
│     │ │ ```                                 │ │                          │
│     │ │ ...numexpr.evaluate(text)...       │ │                          │
│     │ │ ```output                           │ │                          │
│     │ │ ${Output of running the code}      │ │                          │
│     │ │ ```                                 │ │                          │
│     │ │ Answer: ${Answer}                  │ │                          │
│     │ └────────────────────────────────────┘ │                          │
│     │                                         │                          │
│     │ ACTUAL INPUT:                           │                          │
│     │ Question: "15 - 5"                     │                          │
│     └──────────────┬─────────────────────────┘                          │
│                    │                                                     │
│                    ▼                                                     │
│ 6.2 LLM GENERATES MATH EXPRESSION                                       │
│     ┌────────────────────────────────────────┐                          │
│     │ LLM (GPT-3.5-turbo) Processing         │                          │
│     │                                         │                          │
│     │ OUTPUT:                                 │                          │
│     │ ┌────────────────────────────────────┐ │                          │
│     │ │ ```text                            │ │                          │
│     │ │ 15 - 5                             │ │                          │
│     │ │ ```                                │ │                          │
│     │ └────────────────────────────────────┘ │                          │
│     └──────────────┬─────────────────────────┘                          │
│                    │                                                     │
│                    ▼                                                     │
│ 6.3 NUMEXPR EVALUATION                                                  │
│     ┌────────────────────────────────────────┐                          │
│     │ Python numexpr.evaluate()              │                          │
│     │                                         │                          │
│     │ INPUT: "15 - 5"                        │                          │
│     │                                         │                          │
│     │ EXECUTION:                              │                          │
│     │   numexpr.evaluate("15 - 5")           │                          │
│     │                                         │                          │
│     │ RESULT: 10                             │                          │
│     └──────────────┬─────────────────────────┘                          │
│                    │                                                     │
│ TOOL OUTPUT: "Answer: 10"                                               │
└────────────────────────────┬─────────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ STEP 7: OBSERVATION #2                                                   │
│                                                                          │
│ Observation: "Answer: 10"                                               │
│                                                                          │
│ Agent scratchpad обновляется:                                           │
│   [Previous thoughts and actions...]                                     │
│   Thought: I need to calculate 15 - 5                                   │
│   Action: Calculator                                                     │
│   Action Input: 15 - 5                                                  │
│   Observation: Answer: 10                                               │
└────────────────────────────┬─────────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ STEP 8: FINAL THOUGHT                                                    │
│                                                                          │
│ LLM REASONING:                                                           │
│ "I now have all the information. The user currently has 15 vacation    │
│  days, and if they take 5, they will have 10 left."                    │
│                                                                          │
│ OUTPUT:                                                                  │
│   Thought: "I now know the final answer"                                │
└────────────────────────────┬─────────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ STEP 9: FINAL ANSWER                                                     │
│                                                                          │
│ Final Answer: "If you take 5 vacation days, you will have 10           │
│               vacation days left."                                       │
└────────────────────────────┬─────────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ USER OUTPUT                                                              │
│ "If you take 5 vacation days, you will have 10 vacation days left."    │
└──────────────────────────────────────────────────────────────────────────┘
```

### Примеры математических операций:

Calculator tool поддерживает различные математические выражения через numexpr:

**1. Базовая арифметика:**
```python
Action Input: "15 - 5"        → 10
Action Input: "50000 * 0.15"  → 7500.0 (налог 15%)
Action Input: "(30000 / 30) * 5" → 5000.0 (5 дней зарплаты)
```

**2. Сложные выражения:**
```python
Action Input: "(15 + 12) * 0.5"  → 13.5
Action Input: "365 / 12"         → 30.416666...
```

**3. Операции с процентами:**
```python
Action Input: "50000 + (50000 * 0.1)"  → 55000.0 (зарплата + 10% бонус)
```

### Альтернативный сценарий: Только Calculator

Если вопрос чисто математический без необходимости данных:

```
USER: "What is 365 divided by 12?"
  ↓
THOUGHT: "This is a simple math question"
  ↓
ACTION: Calculator
ACTION INPUT: "365 / 12"
  ↓
OBSERVATION: "Answer: 30.416666666666668"
  ↓
FINAL ANSWER: "365 divided by 12 equals approximately 30.42"
```

### Детали передачи данных:

**Промпты и их последовательность:**

1. **Системный промпт** → LLM (Thought #1)
2. **Generated Pandas code** → Employee Data Tool
3. **Data result** → LLM (Thought #2)
4. **Math expression** → LLMMathChain prompt template
5. **Math prompt** → LLM (expression generation)
6. **Expression** → numexpr.evaluate()
7. **Calculation result** → Agent (Observation #2)
8. **Complete context** → LLM (Final Answer)

**Вовлеченные компоненты:**
- `ChatOpenAI` (gpt-3.5-turbo) - основной LLM (используется 4 раза)
- `LLMMathChain` - специальный chain для математики
- `numexpr` - библиотека для безопасного выполнения математических выражений
- `PythonAstREPLTool` - для получения исходных данных
- `Tool` wrapper - обертка для инструментов

**Количество вызовов LLM:** 4
1. Thought #1 + Action selection (Employee Data)
2. Thought #2 + Action selection (Calculator)
3. Math expression generation (внутри LLMMathChain)
4. Final Answer generation

**Особенности LLMMathChain:**
- Использует специальный промпт для преобразования естественного языка в математические выражения
- Поддерживает `numexpr` синтаксис (безопасная математика без exec)
- Автоматически форматирует результат с "Answer:" префиксом
- Verbose mode показывает промежуточные шаги

---
