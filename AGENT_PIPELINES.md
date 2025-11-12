# GPT-Engineer Agent Pipelines Documentation

Этот документ описывает все возможные схемы работы агента gpt-engineer, включая пайплайны генерации, улучшения кода, и обработки ошибок. Каждый пайплайн включает ASCII-диаграмму, описание шагов, используемые промпты и поток данных.

---

## ОГЛАВЛЕНИЕ

1. [Standard Generation Pipeline](#1-standard-generation-pipeline) - Стандартная генерация кода
2. [Code Improvement Pipeline](#2-code-improvement-pipeline) - Улучшение существующего кода
3. [Clarified Generation Pipeline](#3-clarified-generation-pipeline) - Генерация с уточнением требований
4. [Self-Healing Pipeline](#4-self-healing-pipeline) - Автоматическое исправление ошибок
5. [Lite Generation Pipeline](#5-lite-generation-pipeline) - Упрощенная генерация
6. [Complete Agent Flow](#6-complete-agent-flow) - Общая схема работы агента

---

## 1. STANDARD GENERATION PIPELINE

**Описание:** Базовый пайплайн для генерации нового кода с нуля. Используется когда пользователь не указывает специальные флаги (clarify, lite, improve).

**CLI Команда:**
```bash
gpt-engineer <project_path>
```

**Файлы:**
- `gpt_engineer/applications/cli/main.py:543` - точка входа
- `gpt_engineer/applications/cli/cli_agent.py:152` - метод `init()`
- `gpt_engineer/core/default/steps.py:121` - функция `gen_code()`
- `gpt_engineer/core/default/steps.py:153` - функция `gen_entrypoint()`
- `gpt_engineer/core/default/steps.py:205` - функция `execute_entrypoint()`

### ASCII Схема

```
┌─────────────────────────────────────────────────────────────────────┐
│                         USER INPUT PHASE                            │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
                    ┌─────────────────────────┐
                    │  Load User Prompt       │
                    │  (from file or stdin)   │
                    └─────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    CODE GENERATION PHASE                            │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────┐
                    │  gen_code()              │
                    │                          │
                    │  System Prompt:          │
                    │  roadmap +               │
                    │  generate(file_format) + │
                    │  philosophy              │
                    │                          │
                    │  User Prompt:            │
                    │  user's request          │
                    └──────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────┐
                    │  AI Response             │
                    │  (generated code files)  │
                    └──────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────┐
                    │  chat_to_files_dict()    │
                    │  Parse AI response       │
                    │  Extract files from      │
                    │  FILENAME + ``` blocks   │
                    └──────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────┐
                    │  FilesDict               │
                    │  {filename: code}        │
                    └──────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    ENTRYPOINT GENERATION PHASE                      │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────┐
                    │  gen_entrypoint()        │
                    │                          │
                    │  System Prompt:          │
                    │  entrypoint preprompt    │
                    │                          │
                    │  User Prompt:            │
                    │  "Make a unix script     │
                    │   that installs deps     │
                    │   and runs code"         │
                    │  + files_dict.to_chat()  │
                    └──────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────┐
                    │  AI Response             │
                    │  (shell commands in      │
                    │   code blocks)           │
                    └──────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────┐
                    │  Extract commands        │
                    │  Create run.sh           │
                    └──────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────┐
                    │  Merge FilesDict         │
                    │  code + run.sh           │
                    └──────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      EXECUTION PHASE                                │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────┐
                    │  execute_entrypoint()    │
                    │                          │
                    │  Ask user confirmation:  │
                    │  "Execute this code?"    │
                    └──────────────────────────┘
                                  │
                         ┌────────┴────────┐
                         │                 │
                    User says YES     User says NO
                         │                 │
                         ▼                 ▼
            ┌────────────────────┐  ┌──────────────┐
            │ Upload files to    │  │ Skip         │
            │ execution env      │  │ execution    │
            │ Run: bash run.sh   │  └──────────────┘
            └────────────────────┘
                         │
                         ▼
            ┌────────────────────────┐
            │ Code executes          │
            │ (user sees output)     │
            └────────────────────────┘
                         │
                         ▼
            ┌────────────────────────┐
            │ Return FilesDict       │
            └────────────────────────┘
```

### Детальный поток данных

#### Шаг 1: Подготовка системного промпта
```python
system_prompt = (
    roadmap +  # "You will get instructions for code to write..."
    generate.replace("FILE_FORMAT", file_format) +  # Full generation instructions
    "\nUseful to know:\n" +
    philosophy  # Best practices
)
```

#### Шаг 2: Отправка в AI
```python
messages = [
    SystemMessage(content=system_prompt),
    HumanMessage(content=user_prompt)
]
ai_response = ai.start(messages)
```

#### Шаг 3: Парсинг ответа
AI возвращает текст в формате:
```
src/main.py
```python
def main():
    print("Hello")
```

requirements.txt
```
flask==2.0.0
```
```

Парсер извлекает файлы:
```python
{
    "src/main.py": "def main():\n    print(\"Hello\")",
    "requirements.txt": "flask==2.0.0"
}
```

#### Шаг 4: Генерация entrypoint
AI получает:
- System: entrypoint preprompt
- User: дефолтный запрос + содержимое всех файлов

AI возвращает:
```bash
```
pip install -r requirements.txt
python src/main.py
```
```

Извлекается в `run.sh`

#### Шаг 5: Выполнение (опционально)
- Показывает run.sh пользователю
- Спрашивает подтверждение
- При согласии выполняет bash run.sh

### Используемые промпты

1. **roadmap** - базовая инструкция
2. **generate** - детальные инструкции генерации
3. **file_format** - формат вывода файлов
4. **philosophy** - стиль кодирования
5. **entrypoint** - инструкции для run.sh
6. **Default entrypoint user prompt** - "Make a unix script that installs dependencies..."

---
