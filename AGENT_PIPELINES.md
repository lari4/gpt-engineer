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

## 2. CODE IMPROVEMENT PIPELINE

**Описание:** Пайплайн для улучшения существующего кода. Использует формат unified git diff для внесения изменений. Включает цикл исправления некорректных диффов.

**CLI Команда:**
```bash
gpt-engineer <project_path> --improve
```

**Файлы:**
- `gpt_engineer/applications/cli/main.py:516` - точка входа для improve mode
- `gpt_engineer/applications/cli/cli_agent.py:185` - метод `improve()`
- `gpt_engineer/core/default/steps.py:271` - функция `improve_fn()`
- `gpt_engineer/core/default/steps.py:315` - функция `_improve_loop()`
- `gpt_engineer/core/default/steps.py:341` - функция `salvage_correct_hunks()`

### ASCII Схема

```
┌─────────────────────────────────────────────────────────────────────┐
│                     FILE SELECTION PHASE                            │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────┐
                    │  FileSelector            │
                    │  Interactive selection   │
                    │  of files to modify      │
                    │  (or TOML file)          │
                    └──────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────┐
                    │  Optional: Linting       │
                    │  Apply linter to files   │
                    └──────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────┐
                    │  files_dict_before       │
                    │  {filename: old_code}    │
                    └──────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    IMPROVEMENT REQUEST PHASE                        │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────┐
                    │  Load Improvement Prompt │
                    │  "How do you want to     │
                    │   improve the app?"      │
                    └──────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      AI IMPROVEMENT PHASE                           │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────────┐
                    │  improve_fn()                │
                    │                              │
                    │  Build message sequence:     │
                    │                              │
                    │  [1] SystemMessage:          │
                    │      roadmap +               │
                    │      improve(file_format_    │
                    │        diff) + philosophy    │
                    │                              │
                    │  [2] HumanMessage:           │
                    │      files_dict.to_chat()    │
                    │      (all existing files)    │
                    │                              │
                    │  [3] HumanMessage:           │
                    │      user improvement        │
                    │      request                 │
                    └──────────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────────┐
                    │  AI Response                 │
                    │  (unified git diffs)         │
                    └──────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     DIFF VALIDATION LOOP                            │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────────┐
                    │  salvage_correct_hunks()     │
                    │                              │
                    │  1. Parse diffs with regex   │
                    │  2. Validate each diff:      │
                    │     - Check if new file or   │
                    │       existing file          │
                    │     - For existing: validate │
                    │       hunk matches code      │
                    │     - Correct line numbers   │
                    │  3. Collect errors           │
                    └──────────────────────────────┘
                                  │
                         ┌────────┴────────┐
                         │                 │
                   Errors found?      No errors
                         │                 │
                         YES               ▼
                         │        ┌─────────────────┐
                         │        │ Apply diffs     │
                         │        │ Return files    │
                         │        └─────────────────┘
                         ▼
            ┌────────────────────────────┐
            │ Retry Loop                 │
            │ (MAX_EDIT_REFINEMENT_      │
            │  STEPS = 3)                │
            │                            │
            │ Add HumanMessage:          │
            │ "Some diffs were not on    │
            │  requested format or code  │
            │  not found. Details:       │
            │  {errors}                  │
            │  Only rewrite problematic  │
            │  diffs..."                 │
            └────────────────────────────┘
                         │
                         ▼
            ┌────────────────────────────┐
            │ AI next() call             │
            │ Get corrected diffs        │
            └────────────────────────────┘
                         │
                         │ Loop back to validation
                         ▼
            (Repeat until no errors or max retries)
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    REVIEW AND APPLY PHASE                           │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────────┐
                    │  compare()                   │
                    │  Show colored diff to user   │
                    │  (files_dict_before vs       │
                    │   files_dict_after)          │
                    └──────────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────────┐
                    │  User Confirmation           │
                    │  "Apply these changes?"      │
                    └──────────────────────────────┘
                                  │
                         ┌────────┴────────┐
                         │                 │
                    User YES          User NO
                         │                 │
                         ▼                 ▼
            ┌─────────────────────┐ ┌──────────────┐
            │ Apply changes       │ │ Discard      │
            │ files_dict_after    │ │ Keep old     │
            └─────────────────────┘ └──────────────┘
```

### Детальный поток данных

#### Шаг 1: Подготовка системного промпта
```python
system_prompt = (
    roadmap +  # "You will get instructions for code to write..."
    improve.replace("FILE_FORMAT", file_format_diff) +  # Git diff instructions
    "\nUseful to know:\n" +
    philosophy  # Best practices
)
```

#### Шаг 2: Формирование сообщений
```python
messages = [
    SystemMessage(content=system_prompt),
    HumanMessage(content=files_dict.to_chat()),  # All existing files
    HumanMessage(content=user_improvement_prompt)
]
```

Пример files_dict.to_chat():
```
src/main.py
```
def hello():
    print("old version")
```

src/utils.py
```
def helper():
    return 42
```
```

#### Шаг 3: AI возвращает диффы
```diff
--- src/main.py
+++ src/main.py
@@ -1,2 +1,3 @@
 def hello():
-    print("old version")
+    print("new version")
+    print("with improvements")
```

#### Шаг 4: Валидация диффов
```python
# Parse diff
diff = parse_diffs(ai_response)

# Validate hunks
for diff in diffs:
    if not diff.is_new_file():
        problems = diff.validate_and_correct(
            file_to_lines_dict(files_dict[diff.filename_pre])
        )
        
        # Problems might be:
        # - "Line 'old version' not found at expected position"
        # - "Hunk context doesn't match"
```

#### Шаг 5: Retry loop (если есть ошибки)
```python
retries = 0
while errors and retries < MAX_EDIT_REFINEMENT_STEPS:
    messages.append(HumanMessage(
        content="Some diffs were not on requested format...\n"
        + "\n".join(errors) + "\n Only rewrite problematic diffs..."
    ))
    messages = ai.next(messages)
    files_dict, errors = salvage_correct_hunks(messages, files_dict)
    retries += 1
```

### Используемые промпты

1. **roadmap** - базовая инструкция
2. **improve** - инструкции для модификации через git diff
3. **file_format_diff** - детальные правила unified diff
4. **philosophy** - стиль кодирования
5. **Diff refinement error prompt** (динамический) - исправление некорректных диффов

### Ключевые особенности

1. **Diff Validation**: Строгая валидация каждого hunk перед применением
2. **Error Recovery**: До 3 попыток исправить некорректные диффы
3. **Line Number Correction**: Автоматическая корректировка номеров строк
4. **User Review**: Показ изменений и запрос подтверждения

---
