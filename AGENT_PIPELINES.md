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
