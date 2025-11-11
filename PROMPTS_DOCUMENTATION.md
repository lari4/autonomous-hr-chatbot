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
