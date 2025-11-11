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
