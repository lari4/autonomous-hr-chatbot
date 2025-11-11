# Документация AI Промптов
## Autonomous HR Chatbot

Данный документ содержит полное описание всех AI промптов, используемых в приложении Autonomous HR Chatbot.

---

## 1. Системные Промпты (System Prompts)

### 1.1 Основной системный промпт HR ассистента

**Назначение:**
Это главная системная инструкция, которая определяет роль и поведение AI агента. Промпт устанавливает, что AI является дружелюбным HR ассистентом, предназначенным для помощи пользователям в вопросах, связанных с человеческими ресурсами. Промпт также информирует агента о наличии доступных инструментов для выполнения задач.

**Файлы:**
- `hr_agent_backend_local.py:98`
- `hr_agent_backend_azure.py:123`
- `hr-agent-code-jupyter-notebook.ipynb` (Cell-1)

**Промпт:**
```python
'You are friendly HR assistant. You are tasked to assist the current user: {user} on questions related to HR. You have access to the following tools:'
```

**Параметры:**
- `{user}` - динамически подставляемое имя текущего пользователя

**Использование:**
Этот промпт используется как `agent_kwargs={'prefix': ...}` при инициализации Zero-Shot React агента в LangChain. Он формирует начальную часть системного промпта, к которому затем добавляются описания инструментов и инструкции по формату взаимодействия.

---

## 2. Промпты Инструментов (Tool Prompts)

Промпты инструментов описывают назначение и способ использования каждого инструмента, доступного AI агенту. Они включают примеры взаимодействия для демонстрации правильного использования.

### 2.1 Timekeeping Policies Tool

**Назначение:**
Этот промпт инструктирует AI агента, когда и как использовать инструмент для работы с политиками учета рабочего времени. Инструмент используется для ответов на вопросы о политиках отпусков, больничных, отгулов и других аспектов учета времени сотрудников. Промпт включает пример диалога для демонстрации правильного паттерна использования инструмента.

**Файлы:**
- `hr_agent_backend_local.py:65-73`
- `hr_agent_backend_azure.py:90-98`
- `hr-agent-code-jupyter-notebook.ipynb` (Cell-1)

**Промпт:**
```python
"""
Useful for when you need to answer questions about employee timekeeping policies.

<user>: What is the policy on unused vacation leave?
<assistant>: I need to check the timekeeping policies to answer this question.
<assistant>: Action: Timekeeping Policies
<assistant>: Action Input: Vacation Leave Policy - Unused Leave
"""
```

**Использование:**
- Инструмент подключается к векторной базе данных Pinecone, которая содержит эмбеддинги документа `hr_policy.txt`
- При вызове инструмента выполняется семантический поиск по базе знаний
- Возвращает наиболее релевантные фрагменты политик для ответа на вопрос
- Используется модель эмбеддингов OpenAI `text-embedding-ada-002`

---

### 2.2 Employee Data Tool

**Назначение:**
Этот промпт инструктирует AI агента использовать Python Pandas для выполнения операций с данными сотрудников, хранящимися в DataFrame 'df'. Промпт динамически включает список всех доступных колонок в DataFrame и предоставляет пример использования для извлечения персональных данных сотрудника (например, остатка больничных дней). Это критически важный инструмент для ответов на вопросы о статусе сотрудников, балансе отпусков, зарплате и других персональных данных.

**Файлы:**
- `hr_agent_backend_local.py:78-86`
- `hr_agent_backend_azure.py:103-111`
- `hr-agent-code-jupyter-notebook.ipynb` (Cell-1)

**Промпт:**
```python
f"""
Useful for when you need to answer questions about employee data stored in pandas dataframe 'df'.
Run python pandas operations on 'df' to help you get the right answer.
'df' has the following columns: {df_columns}

<user>: How many Sick Leave do I have left?
<assistant>: df[df['name'] == '{user}']['sick_leave']
<assistant>: You have n sick leaves left.
"""
```

**Параметры:**
- `{df_columns}` - динамически подставляемый список колонок DataFrame: `['employee_id', 'name', 'position', 'organizational_unit', 'rank', 'hire_date', 'regularization_date', 'vacation_leave', 'sick_leave', 'basic_pay_in_php', 'employment_status', 'supervisor']`
- `{user}` - имя текущего пользователя для фильтрации данных

**Использование:**
- Инструмент использует LangChain's `PythonAstREPLTool` для безопасного выполнения Python кода
- DataFrame 'df' загружается из файла `employee_data.csv`
- Поддерживает все операции Pandas: фильтрация, агрегация, вычисления
- Пример запросов: остаток отпусков, информация о зарплате, данные супервайзера, статус работы

---

### 2.3 Calculator Tool

**Назначение:**
Простой промпт, который указывает AI агенту использовать калькулятор для выполнения математических операций и арифметических вычислений. Этот инструмент используется, когда необходимо выполнить точные расчеты, например, вычислить остаток дней отпуска после использования, рассчитать пропорциональную зарплату или выполнить другие численные операции.

**Файлы:**
- `hr_agent_backend_local.py:91-93`
- `hr_agent_backend_azure.py:116-118`
- `hr-agent-code-jupyter-notebook.ipynb` (Cell-1)

**Промпт:**
```python
f"""
Useful when you need to do math operations or arithmetic.
"""
```

**Использование:**
- Инструмент использует LangChain's `LLMMathChain` для выполнения математических вычислений
- Поддерживает базовые арифметические операции: сложение, вычитание, умножение, деление
- Поддерживает сложные математические выражения через Python
- Примеры использования: расчет процентов, вычисление разницы дат, арифметика с балансом отпусков

---

## 3. Шаблон Zero-Shot React Agent

### 3.1 Полный промпт агента (Generated Template)

**Назначение:**
Это полный сгенерированный LangChain промпт, который объединяет системный промпт, описания всех инструментов и инструкции по формату взаимодействия. Промпт определяет паттерн ReAct (Reasoning + Acting) для пошагового решения задач: агент думает (Thought), выбирает действие (Action), получает результат (Observation) и повторяет цикл до получения окончательного ответа (Final Answer).

**Файлы:**
- `hr-agent-code-jupyter-notebook.ipynb` (Cell-2, вывод команды `print(agent.agent.llm_chain.prompt.template)`)

**Промпт:**
```text
You are friendly HR assistant. You are tasked to assist the current user: Alexander Verdad on questions related to HR. You have access to the following tools:

Timekeeping Policies:
        Useful for when you need to answer questions about employee timekeeping policies.

Employee Data:
        Useful for when you need to answer questions about employee data stored in pandas dataframe 'df'.
        Run python pandas operations on 'df' to help you get the right answer.
        'df' has the following columns: ['employee_id', 'name', 'position', 'organizational_unit', 'rank', 'hire_date', 'regularization_date', 'vacation_leave', 'sick_leave', 'basic_pay_in_php', 'employment_status', 'supervisor']

        <user>: How many Sick Leave do I have left?
        <assistant>: df[df['name'] == 'Alexander Verdad']['sick_leave']
        <assistant>: You have n sick leaves left.

Calculator:
        Useful when you need to do math operations or arithmetic.


Use the following format:

Question: the input question you must answer
Thought: you should always think about what to do
Action: the action to take, should be one of [Timekeeping Policies, Employee Data, Calculator]
Action Input: the input to the action
Observation: the result of the action
... (this Thought/Action/Action Input/Observation can repeat N times)
Thought: I now know the final answer
Final Answer: the final answer to the original input question

Begin!

Question: {input}
Thought:{agent_scratchpad}
```

**Параметры:**
- `{input}` - вопрос пользователя
- `{agent_scratchpad}` - история рассуждений и действий агента (заполняется автоматически)

**Использование:**
- Этот промпт автоматически генерируется LangChain при создании `initialize_agent` с типом `AgentType.ZERO_SHOT_REACT_DESCRIPTION`
- Определяет формат работы агента: циклический процесс мышления и действия
- Агент следует этому формату для структурированного решения задач
- Примеры действий видны в логах выполнения агента

---

## 4. База Знаний (Knowledge Base)

### 4.1 HR Policy Document

**Назначение:**
Это основной документ с политиками компании, который служит источником знаний для инструмента "Timekeeping Policies". Документ содержит детальные правила и регламенты по всем видам отпусков и посещаемости. Текст документа разбивается на фрагменты (chunks), преобразуется в векторные эмбеддинги с помощью OpenAI и сохраняется в векторную базу данных Pinecone для семантического поиска.

**Файлы:**
- `hr_policy.txt` (282 строки)
- Обработка: `store_embeddings_in_pinecone.ipynb`

**Структура документа:**

#### Часть 1: HR Policy Manual - Leave Policy (строки 1-163)

Содержит политики по следующим типам отпусков:

**A. Vacation Leave (Отпуск)**
- Право и начисление: 1.25 дня в месяц (15 дней в год)
- Подача заявок через Employee Self Service портал
- Неиспользованный отпуск переносится (максимум 30 дней)
- Можно обналичить по формуле: (базовая зарплата / 30) × неиспользованные дни
- Не доступно на испытательном сроке

**B. Sick Leave (Больничный)**
- Право и начисление: 1.25 дня в месяц (15 дней в год)
- Требуется медицинская справка для более 2 дней подряд
- Неиспользованный больничный переносится (максимум 30 дней)
- Нельзя обналичить
- На испытательном сроке: 0.625 дней в месяц (7.5 дней в год)

**C. Service Incentive Leave (Поощрительный отпуск)**
- 5 дней в год после года работы
- Не переносится на следующий год
- Можно обналичить

**D. Paternity Leave (Отпуск по уходу за ребенком - отец)**
- 7 дней на каждого ребенка (до 4 раз)
- Доступно на испытательном сроке

**E. Maternity Leave (Отпуск по уходу за ребенком - мать)**
- 105 дней на каждого ребенка (до 2 раз)
- Доступно на испытательном сроке

**F. Violence Against Women Leave**
- 10 дней в год для жертв насилия
- Доступно на испытательном сроке

**G. Single Parent Leave**
- 7 дней в год для родителей-одиночек
- Доступно на испытательном сроке

**H. Solo Parent Leave**
- 7 дней в год
- Требуется Solo Parent ID Card

**I. Parental Leave for Solo Parents**
- 7 дней в год дополнительно

**J. Special Leave Benefit for Women**
- 2 месяца для гинекологических операций

**K. Bereavement Leave**
- 3 дня на похороны близких родственников

**L. Emergency Leave**
- По усмотрению при чрезвычайных обстоятельствах

#### Часть 2: HR Policy Manual - Attendance Policy (строки 164-282)

Содержит политики по:

**A. Official Business (Официальные командировки)**
- Длительность = длительности командировки
- Доступно на испытательном сроке

**B. Overtime (Сверхурочные)**
- Оплачивается с премией согласно политике компании
- Доступно на испытательном сроке

**C. Training and Development Leave**
- Длительность = длительности тренинга
- Доступно на испытательном сроке

**D. Work From Home (Удаленная работа)**
- Длительность согласуется с супервайзером
- Доступно на испытательном сроке

**E. Leave Without Pay**
- До 30 дней в году при исчерпании оплачиваемых отпусков
- Доступно на испытательном сроке

**Технические детали обработки:**
```python
# Параметры разбиения текста (из store_embeddings_in_pinecone.ipynb)
chunk_size = 400  # символов в каждом фрагменте
chunk_overlap = 20  # символов пересечения между фрагментами

# Модель эмбеддингов
model = "text-embedding-ada-002"  # OpenAI

# Векторная база данных
database = "Pinecone"
index_name = "langchain-demo"
```

**Использование:**
- Документ загружается с помощью `TextLoader`
- Разбивается на фрагменты с помощью `CharacterTextSplitter`
- Каждый фрагмент преобразуется в векторный эмбеддинг
- Сохраняется в Pinecone для быстрого семантического поиска
- При запросе пользователя выполняется поиск релевантных фрагментов
- Найденные фрагменты используются как контекст для ответа AI агента

---

## 5. Архитектура и Поток Промптов

### 5.1 Общая Архитектура Системы

Приложение использует **LangChain framework** с архитектурой **Zero-Shot ReAct Agent** для создания автономного HR ассистента. Вот как промпты взаимодействуют в системе:

```
┌─────────────────────────────────────────────────────────────────┐
│                        ПОЛЬЗОВАТЕЛЬ                              │
│                     "How many sick leaves                        │
│                      do I have left?"                           │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                    ZERO-SHOT REACT AGENT                         │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ Системный промпт:                                           │ │
│  │ "You are friendly HR assistant..."                         │ │
│  └────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ Промпты инструментов:                                       │ │
│  │ - Timekeeping Policies                                     │ │
│  │ - Employee Data                                            │ │
│  │ - Calculator                                               │ │
│  └────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ ReAct формат:                                              │ │
│  │ Question → Thought → Action → Observation → Final Answer  │ │
│  └────────────────────────────────────────────────────────────┘ │
└──────────┬──────────────────┬──────────────────┬───────────────┘
           │                  │                  │
           ▼                  ▼                  ▼
    ┌──────────────┐   ┌─────────────┐   ┌────────────┐
    │ Timekeeping  │   │  Employee   │   │ Calculator │
    │  Policies    │   │    Data     │   │            │
    │              │   │             │   │            │
    │ ┌──────────┐ │   │ ┌─────────┐ │   │ ┌────────┐ │
    │ │ Pinecone │ │   │ │ Pandas  │ │   │ │LLMMath │ │
    │ │ Vector   │ │   │ │DataFrame│ │   │ │ Chain  │ │
    │ │   DB     │ │   │ │   'df'  │ │   │ │        │ │
    │ └──────────┘ │   │ └─────────┘ │   │ └────────┘ │
    │      ▲       │   │      ▲      │   │            │
    │      │       │   │      │      │   │            │
    │ ┌────┴─────┐ │   │ ┌────┴────┐ │   │            │
    │ │hr_policy │ │   │ │employee │ │   │            │
    │ │   .txt   │ │   │ │_data.csv│ │   │            │
    │ │embeddings│ │   │ └─────────┘ │   │            │
    │ └──────────┘ │   │             │   │            │
    └──────────────┘   └─────────────┘   └────────────┘
```

### 5.2 Поток Выполнения Запроса

**Пример: "How many sick leaves do I have left?"**

1. **Инициализация:**
   - Загружается системный промпт с именем пользователя
   - Загружаются промпты всех 3 инструментов
   - Создается полный шаблон ReAct агента

2. **Получение запроса:**
   - Вопрос: "How many sick leaves do I have left?"
   - Вставляется в переменную `{input}` шаблона агента

3. **Цикл ReAct:**

   **Thought 1:** "I need to check the employee data to find out how many sick leaves this user has"

   **Action 1:** Employee Data

   **Action Input 1:** `df[df['name'] == 'Alexander Verdad']['sick_leave']`

   **Observation 1:** 12

   **Thought 2:** "I now know the final answer"

   **Final Answer:** "You have 12 sick leaves left."

4. **Возврат результата пользователю**

### 5.3 Технический Стек

- **LLM Model:** OpenAI GPT-3.5-turbo (локальная версия) / Azure OpenAI (Azure версия)
- **Framework:** LangChain
- **Agent Type:** Zero-Shot ReAct Description
- **Tools:**
  - `VectorDBQA` (Pinecone + OpenAI Embeddings)
  - `PythonAstREPLTool` (Pandas операции)
  - `LLMMathChain` (математические операции)
- **Vector Database:** Pinecone
- **Embedding Model:** text-embedding-ada-002 (OpenAI)
- **Backend:** Gradio (UI), Python (логика)

### 5.4 Файловая Структура Промптов

```
autonomous-hr-chatbot/
├── hr_agent_backend_local.py          # Основной backend (локальный OpenAI)
│   ├── Системный промпт (строка 98)
│   ├── Timekeeping Policies промпт (строки 65-73)
│   ├── Employee Data промпт (строки 78-86)
│   └── Calculator промпт (строки 91-93)
│
├── hr_agent_backend_azure.py          # Backend для Azure OpenAI
│   ├── Системный промпт (строка 123)
│   ├── Timekeeping Policies промпт (строки 90-98)
│   ├── Employee Data промпт (строки 103-111)
│   └── Calculator промпт (строки 116-118)
│
├── hr_policy.txt                      # База знаний (282 строки)
│   ├── Leave Policy (строки 1-163)
│   └── Attendance Policy (строки 164-282)
│
├── employee_data.csv                  # Данные сотрудников для Pandas
│
├── store_embeddings_in_pinecone.ipynb # Скрипт для создания эмбеддингов
│
└── hr-agent-code-jupyter-notebook.ipynb # Демонстрация полного промпта
    └── Вывод полного шаблона агента (Cell-2)
```

---

## 6. Рекомендации по Улучшению Промптов

### 6.1 Текущие Недостатки

1. **Системный промпт слишком короткий:**
   - Не хватает инструкций по тону общения
   - Нет указаний по обработке конфиденциальной информации
   - Отсутствуют правила эскалации сложных вопросов

2. **Примеры в промптах инструментов:**
   - Employee Data содержит только один пример
   - Timekeeping Policies имеет базовый пример
   - Calculator не имеет примеров вообще

3. **Отсутствие обработки ошибок:**
   - Нет инструкций, что делать при неполных данных
   - Нет промптов для обработки неоднозначных запросов

### 6.2 Предложения по Улучшению

1. **Расширить системный промпт:**
```python
'You are a friendly and professional HR assistant. You are tasked to assist the current user: {user} on questions related to HR policies, leave balances, and employee information.

Guidelines:
- Always maintain confidentiality and data privacy
- Provide accurate information based on company policies
- If you are unsure about something, clearly state it
- Be empathetic when dealing with sensitive topics
- Escalate complex legal or medical questions to HR department
- Use clear, professional language

You have access to the following tools:'
```

2. **Добавить больше примеров в промпты инструментов:**
   - Разнообразные сценарии использования
   - Примеры edge cases
   - Примеры комбинированного использования инструментов

3. **Добавить промпт для обработки ошибок:**
```python
'When you encounter an error or missing data:
1. Politely inform the user about the issue
2. Suggest alternative approaches or workarounds
3. Provide contact information for direct HR support if needed'
```

---

## 7. Заключение

Данная документация описывает все AI промпты, используемые в Autonomous HR Chatbot. Система использует модульный подход с разделением промптов на:

1. **Системные промпты** - определяют роль и поведение агента
2. **Промпты инструментов** - инструктируют использование специфических функций
3. **Шаблон агента** - определяет формат взаимодействия (ReAct)
4. **База знаний** - содержит контент для ответов на вопросы

Такая архитектура обеспечивает гибкость, масштабируемость и легкость модификации отдельных компонентов системы без влияния на остальные части.

**Версия документа:** 1.0
**Дата создания:** 2025-11-11
**Автор:** Claude AI Assistant

---
