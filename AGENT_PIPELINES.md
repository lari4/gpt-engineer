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

## 3. CLARIFIED GENERATION PIPELINE

**Описание:** Интерактивный пайплайн с фазой уточнения требований перед генерацией кода. AI задает уточняющие вопросы пользователю, чтобы лучше понять задачу.

**CLI Команда:**
```bash
gpt-engineer <project_path> --clarify
```

**Файлы:**
- `gpt_engineer/tools/custom_steps.py:122` - функция `clarified_gen()`
- `gpt_engineer/core/default/steps.py:121` - используется `gen_code()` после clarification

### ASCII Схема

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CLARIFICATION PHASE                              │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────────┐
                    │  Load User Prompt            │
                    │  (initial request)           │
                    └──────────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────────┐
                    │  Build Initial Messages:     │
                    │                              │
                    │  [SystemMessage]:            │
                    │    clarify preprompt         │
                    │                              │
                    │  [HumanMessage]:             │
                    │    user's initial prompt     │
                    └──────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  INTERACTIVE CLARIFICATION LOOP                     │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────────┐
                    │  ai.next(messages)           │
                    │  AI analyzes prompt          │
                    └──────────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────────┐
                    │  AI Response                 │
                    └──────────────────────────────┘
                                  │
                         ┌────────┴─────────┐
                         │                  │
            Contains "nothing to      Has clarifying
            clarify" (case-insens)    question
                         │                  │
                         ▼                  ▼
            ┌─────────────────┐   ┌────────────────────┐
            │ Exit loop       │   │ Show question to   │
            │ Proceed to      │   │ user               │
            │ generation      │   │                    │
            └─────────────────┘   │ Prompt: "(answer   │
                                  │  in text, or 'c'   │
                                  │  to move on)"      │
                                  └────────────────────┘
                                           │
                                           ▼
                              ┌────────────────────────┐
                              │ User Input             │
                              └────────────────────────┘
                                           │
                              ┌────────────┴────────────┐
                              │                         │
                         User enters         User presses Enter
                         answer              or types "c"
                              │                         │
                              │                         ▼
                              │            ┌──────────────────────┐
                              │            │ Add to messages:     │
                              │            │ "Make your own       │
                              │            │  assumptions and     │
                              │            │  state them          │
                              │            │  explicitly"         │
                              │            └──────────────────────┘
                              │                         │
                              ▼                         ▼
                    ┌───────────────────────────────────────┐
                    │ Add user input to messages:           │
                    │ [HumanMessage]:                       │
                    │   user_input + "\n\n                  │
                    │   Is anything else unclear?           │
                    │   If yes, ask another question.\n     │
                    │   Otherwise state:                    │
                    │   'Nothing to clarify'"               │
                    └───────────────────────────────────────┘
                                           │
                                           │ Loop back
                                           ▼
                              (Repeat until "nothing to clarify")
                                           │
                                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    CODE GENERATION PHASE                            │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────────┐
                    │  Transform Messages:         │
                    │                              │
                    │  [1] Replace first system    │
                    │      message (clarify) with: │
                    │      roadmap +               │
                    │      generate(file_format) + │
                    │      philosophy              │
                    │                              │
                    │  [2] Keep all clarification  │
                    │      dialog messages         │
                    │      (questions & answers)   │
                    │                              │
                    │  [3] Add final message:      │
                    │      generate prompt with    │
                    │      file_format             │
                    └──────────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────────┐
                    │  ai.next(messages)           │
                    │  Generate code               │
                    └──────────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────────┐
                    │  AI Response                 │
                    │  (generated code files)      │
                    └──────────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────────┐
                    │  chat_to_files_dict()        │
                    │  Parse and extract files     │
                    └──────────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────────┐
                    │  Return FilesDict            │
                    └──────────────────────────────┘
                                  │
                                  ▼
                    (Continue with standard generation:
                     gen_entrypoint -> execute_entrypoint)
```

### Детальный поток данных

#### Фаза 1: Начальная clarification
```python
messages = [
    SystemMessage(content=preprompts["clarify"]),
    # clarify: "Given some instructions, determine if anything 
    #          needs to be clarified..."
]

messages = ai.next(messages, user_input=user_prompt)
```

#### Фаза 2: Интерактивный диалог
```python
while True:
    ai_response = messages[-1].content.strip()
    
    # Exit condition
    if "nothing to clarify" in ai_response.lower():
        break
    
    # Show question and get user input
    user_input = input("(answer in text, or 'c' to move on)\n")
    
    # Handle skip
    if not user_input or user_input == "c":
        messages = ai.next(
            messages,
            "Make your own assumptions and state them explicitly"
        )
        continue
    
    # Add continuation prompt
    user_input += """
        \n\n
        Is anything else unclear? If yes, ask another question.\n
        Otherwise state: "Nothing to clarify"
        """
    
    messages = ai.next(messages, user_input)
```

**Пример диалога:**

```
User: Create a snake game in Python

AI: What control scheme would you like?
    - Arrow keys
    - WASD
    - Mouse
    
User: Arrow keys

AI: Should the snake wrap around the edges or die when hitting walls?

User: Die when hitting walls

AI: Nothing to clarify.
```

#### Фаза 3: Трансформация в generation
```python
# Replace system message
messages = [
    SystemMessage(content=setup_sys_prompt(preprompts))
    # = roadmap + generate(file_format) + philosophy
] + messages[1:]  # Keep all clarification dialog

# Add final generation prompt
messages = ai.next(
    messages,
    preprompts["generate"].replace("FILE_FORMAT", preprompts["file_format"])
)
```

**Финальная структура сообщений:**
```python
[
    SystemMessage: roadmap + generate + philosophy
    HumanMessage: "Create a snake game in Python"
    AIMessage: "What control scheme..."
    HumanMessage: "Arrow keys\n\nIs anything else unclear?..."
    AIMessage: "Should the snake wrap..."
    HumanMessage: "Die when hitting walls\n\nIs anything else unclear?..."
    AIMessage: "Nothing to clarify"
    HumanMessage: generate prompt with file_format
    AIMessage: <generated code>
]
```

### Используемые промпты

1. **clarify** - начальная инструкция для уточнения
2. **Assumption prompt** (динамический) - "Make your own assumptions..."
3. **Continuation prompt** (динамический) - "Is anything else unclear?..."
4. **roadmap** - для финальной генерации
5. **generate** - для финальной генерации
6. **file_format** - для финальной генерации
7. **philosophy** - для финальной генерации

### Ключевые особенности

1. **Interactive Dialog**: Многоуровневый диалог с пользователем
2. **Context Preservation**: Весь диалог сохраняется для финальной генерации
3. **Skip Option**: Пользователь может пропустить вопросы (c/Enter)
4. **Automatic Assumptions**: AI делает предположения при пропуске
5. **Smooth Transition**: Плавный переход от clarification к generation

---

## 4. SELF-HEALING PIPELINE

**Описание:** Автоматический пайплайн исправления ошибок выполнения. После генерации кода система выполняет его, и если возникают ошибки, автоматически отправляет их в AI для исправления. Процесс повторяется до успешного выполнения или достижения лимита попыток.

**CLI Команда:**
```bash
gpt-engineer <project_path> --self-heal
```

**Файлы:**
- `gpt_engineer/tools/custom_steps.py:40` - функция `self_heal()`
- `gpt_engineer/core/default/steps.py:271` - использует `improve_fn()` для исправлений

### ASCII Схема

```
┌─────────────────────────────────────────────────────────────────────┐
│                    INITIAL CODE GENERATION                          │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
                    (Standard generation pipeline:
                     gen_code -> gen_entrypoint)
                                  │
                                  ▼
                    ┌──────────────────────────────┐
                    │  FilesDict with run.sh       │
                    └──────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    SELF-HEALING LOOP                                │
│                    (MAX_SELF_HEAL_ATTEMPTS = 10)                    │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────────┐
                    │  Check: run.sh exists?       │
                    └──────────────────────────────┘
                                  │
                         ┌────────┴────────┐
                         │                 │
                      YES               NO
                         │                 │
                         ▼                 ▼
                    Continue      ┌──────────────────┐
                                  │ Raise            │
                                  │ FileNotFoundError│
                                  └──────────────────┘
                         │
                         ▼
            ┌────────────────────────────┐
            │ Attempt Counter = 0        │
            └────────────────────────────┘
                         │
                         ▼
            ┌────────────────────────────────────┐
            │ EXECUTION ATTEMPT                  │
            └────────────────────────────────────┘
                         │
                         ▼
            ┌────────────────────────────────────┐
            │ execution_env.upload(files_dict)   │
            │ Upload all files to exec env       │
            └────────────────────────────────────┘
                         │
                         ▼
            ┌────────────────────────────────────┐
            │ p = execution_env.popen(run.sh)    │
            │ Start process                      │
            └────────────────────────────────────┘
                         │
                         ▼
            ┌────────────────────────────────────┐
            │ stdout, stderr = p.communicate()   │
            │ Wait for completion                │
            └────────────────────────────────────┘
                         │
                         ▼
            ┌────────────────────────────────────┐
            │ Check Return Code                  │
            └────────────────────────────────────┘
                         │
         ┌───────────────┴───────────────┐
         │                               │
    returncode == 0          returncode != 0
    or returncode == 2       and != 2
         │                               │
         ▼                               ▼
    ┌──────────┐         ┌────────────────────────────┐
    │ SUCCESS  │         │ FAILURE - Need to fix      │
    │ Exit     │         │                            │
    │ loop     │         │ Print stdout/stderr        │
    └──────────┘         └────────────────────────────┘
                                        │
                                        ▼
                         ┌────────────────────────────────────┐
                         │ Build Error Prompt:                │
                         │                                    │
                         │ "A program with this               │
                         │  specification was requested:      │
                         │  {original_prompt}                 │
                         │                                    │
                         │  but running it produced the       │
                         │  following output:                 │
                         │  {stdout}                          │
                         │                                    │
                         │  and the following errors:         │
                         │  {stderr}                          │
                         │                                    │
                         │  Please change it so that it       │
                         │  fulfills the requirements."       │
                         └────────────────────────────────────┘
                                        │
                                        ▼
                         ┌────────────────────────────────────┐
                         │ improve_fn()                       │
                         │                                    │
                         │ Uses improvement pipeline with     │
                         │ error prompt to fix code           │
                         │                                    │
                         │ System: roadmap + improve +        │
                         │         philosophy                 │
                         │ User 1: files_dict.to_chat()       │
                         │ User 2: error prompt               │
                         └────────────────────────────────────┘
                                        │
                                        ▼
                         ┌────────────────────────────────────┐
                         │ AI generates diffs to fix errors   │
                         │ Apply diffs to files_dict          │
                         └────────────────────────────────────┘
                                        │
                                        ▼
                         ┌────────────────────────────────────┐
                         │ Increment attempts counter         │
                         └────────────────────────────────────┘
                                        │
                         ┌──────────────┴──────────────┐
                         │                             │
                  attempts < 10                 attempts >= 10
                         │                             │
                         │                             ▼
                         │              ┌────────────────────────┐
                         │              │ Give up                │
                         │              │ Return current         │
                         │              │ files_dict (broken)    │
                         │              └────────────────────────┘
                         │
                         │ Loop back to execution
                         ▼
            (Try executing fixed code again)
```

### Детальный поток данных

#### Шаг 1: Первоначальная генерация
```python
# Standard generation happens first
files_dict = gen_code(ai, prompt, memory, preprompts_holder)
entrypoint = gen_entrypoint(ai, prompt, files_dict, memory, preprompts_holder)
files_dict = {**files_dict, **entrypoint}
```

#### Шаг 2: Первая попытка выполнения
```python
attempts = 0
while attempts < MAX_SELF_HEAL_ATTEMPTS:
    attempts += 1
    
    # Upload and execute
    execution_env.upload(files_dict)
    p = execution_env.popen(files_dict[ENTRYPOINT_FILE])
    stdout_full, stderr_full = p.communicate()
    
    # Check result
    if (p.returncode != 0 and p.returncode != 2):
        # FAILURE - need to fix
        ...
    else:
        # SUCCESS
        break
```

#### Шаг 3: Построение error prompt (при ошибке)
```python
error_prompt = Prompt(
    f"A program with this specification was requested:\n{original_prompt}\n"
    f", but running it produced the following output:\n{stdout_full}\n"
    f" and the following errors:\n{stderr_full}."
    f" Please change it so that it fulfills the requirements."
)
```

**Пример error prompt:**
```
A program with this specification was requested:
Create a Flask web server on port 8000

but running it produced the following output:
Traceback (most recent call last):
  File "main.py", line 1, in <module>
    from flask import Flask
ModuleNotFoundError: No module named 'flask'

and the following errors:


Please change it so that it fulfills the requirements.
```

#### Шаг 4: Вызов improve_fn для исправления
```python
files_dict = improve_fn(
    ai,
    error_prompt,  # Error description as prompt
    files_dict,    # Current (broken) code
    memory,
    preprompts_holder,
    diff_timeout
)
```

AI получает:
- System: roadmap + improve(file_format_diff) + philosophy
- User 1: весь текущий код
- User 2: описание ошибки с просьбой исправить

AI возвращает диффы, например:
```diff
--- requirements.txt
+++ requirements.txt
@@ -0,0 +1 @@
+flask==2.0.0
```

#### Шаг 5: Повторная попытка
Обновленный код снова выполняется, цикл продолжается до успеха или лимита попыток.

### Пример полного цикла

**Attempt 1:**
```
Run: python main.py
Error: ModuleNotFoundError: No module named 'flask'
→ AI adds flask to requirements.txt
```

**Attempt 2:**
```
Run: pip install -r requirements.txt && python main.py
Error: TypeError: Flask() missing required argument 'import_name'
→ AI fixes Flask initialization: Flask(__name__)
```

**Attempt 3:**
```
Run: pip install -r requirements.txt && python main.py
Success: * Running on http://127.0.0.1:8000/
→ Exit loop
```

### Используемые промпты

1. **roadmap** - через improve_fn
2. **improve** - для создания исправлений
3. **file_format_diff** - формат диффов
4. **philosophy** - стиль кодирования
5. **Self-healing error prompt** (динамический) - описание ошибки выполнения

### Ключевые особенности

1. **Automatic Error Recovery**: Полностью автоматическое исправление без участия пользователя
2. **Multiple Attempts**: До 10 попыток исправить код
3. **Full Context**: AI получает полный stdout и stderr для анализа
4. **Uses Improvement Pipeline**: Переиспользует механизм улучшения кода
5. **Iterative Fixing**: Каждая итерация строится на результатах предыдущей

### Ограничения

- Лимит 10 попыток (MAX_SELF_HEAL_ATTEMPTS)
- Не гарантирует успех для сложных ошибок
- Может зациклиться на одной и той же ошибке
- Не обрабатывает бесконечные циклы в коде (нужен timeout)

---

## 5. LITE GENERATION PIPELINE

**Описание:** Упрощенный пайплайн генерации без roadmap и philosophy. Использует только пользовательский промпт и file_format инструкции. Подходит для простых задач или быстрого прототипирования.

**CLI Команда:**
```bash
gpt-engineer <project_path> --lite
```

**Файлы:**
- `gpt_engineer/tools/custom_steps.py:198` - функция `lite_gen()`

### ASCII Схема

```
┌─────────────────────────────────────────────────────────────────────┐
│                    LITE GENERATION PHASE                            │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────────┐
                    │  Load User Prompt            │
                    │  (from file or stdin)        │
                    └──────────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────────┐
                    │  Build Messages:             │
                    │                              │
                    │  INVERTED STRUCTURE:         │
                    │                              │
                    │  System Message:             │
                    │    user_prompt               │
                    │    (NOT preprompts!)         │
                    │                              │
                    │  User Message:               │
                    │    file_format preprompt     │
                    │                              │
                    └──────────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────────┐
                    │  ai.start(                   │
                    │    system=user_prompt,       │
                    │    user=file_format          │
                    │  )                           │
                    └──────────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────────┐
                    │  AI Response                 │
                    │  (generated code files)      │
                    └──────────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────────┐
                    │  chat_to_files_dict()        │
                    │  Parse FILENAME + ```        │
                    │  code blocks                 │
                    └──────────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────────┐
                    │  Return FilesDict            │
                    └──────────────────────────────┘
                                  │
                                  ▼
                    (Continue with standard flow:
                     gen_entrypoint -> execute_entrypoint)
```

### Детальный поток данных

#### Инвертированная структура промптов
```python
# ОБРАТИТЕ ВНИМАНИЕ: это инвертированная структура!
messages = ai.start(
    prompt.to_langchain_content(),  # User prompt as SYSTEM message
    preprompts["file_format"],      # file_format as USER message
    step_name="lite_gen"
)
```

**Стандартный режим:**
```
System: roadmap + generate + philosophy
User: user's request
```

**Lite режим:**
```
System: user's request
User: file_format instructions
```

#### Пример использования

**User Prompt:**
```
Create a simple TODO list web app using Flask
```

**Фактические сообщения в AI:**
```python
[
    SystemMessage(content="Create a simple TODO list web app using Flask"),
    HumanMessage(content="""You will output the content of each file necessary to achieve the goal, including ALL code.
Represent files like so:

FILENAME
```
CODE
```

The following tokens must be replaced like so:
FILENAME is the lowercase combined path and file name including the file extension
CODE is the code in the file

Example representation of a file:

src/hello_world.py
```
print("Hello World")
```

Do not comment on what every file does. Please note that the code should be fully functional. No placeholders.""")
]
```

#### AI Response
```
app.py
```python
from flask import Flask, render_template, request, redirect
import sqlite3

app = Flask(__name__)

@app.route('/')
def index():
    conn = sqlite3.connect('todos.db')
    c = conn.cursor()
    c.execute('SELECT * FROM todos')
    todos = c.fetchall()
    conn.close()
    return render_template('index.html', todos=todos)
    
# ... more code ...
```

templates/index.html
```html
<!DOCTYPE html>
<html>
<head><title>TODO List</title></head>
<body>
    <!-- ... HTML ... -->
</body>
</html>
```

requirements.txt
```
flask==2.0.0
```
```

### Используемые промпты

1. **file_format** - единственный preprompt (как user message)
2. **User's prompt** - используется как system message (!)
3. **entrypoint** - для последующей генерации run.sh (если продолжается стандартный flow)

### Ключевые особенности

1. **Minimal Overhead**: Минимум промптов, быстрая генерация
2. **Inverted Structure**: Уникальная инвертированная структура сообщений
3. **No Best Practices**: Не включает philosophy и roadmap
4. **Simple Tasks**: Лучше всего подходит для простых задач
5. **Same Output Format**: Использует тот же file_format что и стандартный режим

### Отличия от стандартной генерации

| Аспект | Standard Generation | Lite Generation |
|--------|---------------------|-----------------|
| System Message | roadmap + generate + philosophy | user's prompt |
| User Message | user's request | file_format |
| Complexity | Высокая | Низкая |
| Best Practices | Включены (philosophy) | Не включены |
| Detailed Instructions | Да (generate) | Нет |
| Use Case | Сложные проекты | Простые задачи |

### Когда использовать lite mode

✅ **Хорошо для:**
- Простых скриптов
- Быстрого прототипирования
- Одиночных файлов
- Экспериментов

❌ **Не подходит для:**
- Больших проектов
- Когда важна архитектура
- Когда нужны best practices
- Production-ready кода

---

## 6. COMPLETE AGENT FLOW

**Описание:** Общая схема работы CLI агента, показывающая как все пайплайны интегрируются в единую систему.

### Общая архитектура

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CLI ENTRY POINT                             │
│                    gpt_engineer/applications/cli/main.py            │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────────┐
                    │  Parse Arguments             │
                    │  - project_path              │
                    │  - model                     │
                    │  - temperature               │
                    │  - improve / clarify /       │
                    │    lite / self-heal flags    │
                    └──────────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────────┐
                    │  Load Environment            │
                    │  - OPENAI_API_KEY            │
                    │  - ANTHROPIC_API_KEY         │
                    └──────────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────────┐
                    │  Initialize Components       │
                    │  - AI (model + temperature)  │
                    │  - Memory (DiskMemory)       │
                    │  - ExecutionEnv              │
                    │  - PrepromptsHolder          │
                    └──────────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────────┐
                    │  Select Generation Function  │
                    └──────────────────────────────┘
                                  │
         ┌────────────────────────┼────────────────────────┐
         │                        │                        │
         ▼                        ▼                        ▼
    clarify_mode?           lite_mode?              (default)
         │                        │                        │
         ▼                        ▼                        ▼
    clarified_gen()          lite_gen()              gen_code()
                                  │
                                  └────────┬───────────────┘
                                           │
                                           ▼
                    ┌──────────────────────────────┐
                    │  Select Execution Function   │
                    └──────────────────────────────┘
                                  │
                         ┌────────┴────────┐
                         │                 │
                  self_heal_mode?      (default)
                         │                 │
                         ▼                 ▼
                  self_heal()      execute_entrypoint()
                                           │
                                           └────────┬──────┘
                                                    │
                                                    ▼
                    ┌──────────────────────────────────────┐
                    │  Create CliAgent                     │
                    │  agent = CliAgent.with_default_      │
                    │           config(                    │
                    │    memory,                           │
                    │    execution_env,                    │
                    │    ai,                               │
                    │    code_gen_fn=selected_gen_fn,      │
                    │    improve_fn=improve_fn,            │
                    │    process_code_fn=selected_exec_fn, │
                    │    preprompts_holder                 │
                    │  )                                   │
                    └──────────────────────────────────────┘
                                                    │
                                                    ▼
                              ┌─────────────────────────────┐
                              │  Mode Selection             │
                              └─────────────────────────────┘
                                                    │
                         ┌──────────────────────────┴──────────────────┐
                         │                                             │
                         ▼                                             ▼
            ┌────────────────────────┐                    ┌────────────────────────┐
            │  IMPROVE MODE          │                    │  GENERATION MODE       │
            │  (--improve flag)      │                    │  (default)             │
            └────────────────────────┘                    └────────────────────────┘
                         │                                             │
                         ▼                                             ▼
            ┌────────────────────────────┐               ┌────────────────────────┐
            │  FileSelector.ask_for_     │               │  agent.init(prompt)    │
            │    files()                 │               │                        │
            │  User selects files to     │               │  1. code_gen_fn()      │
            │  modify (interactive or    │               │  2. gen_entrypoint()   │
            │  from TOML)                │               │  3. process_code_fn()  │
            └────────────────────────────┘               └────────────────────────┘
                         │                                             │
                         ▼                                             │
            ┌────────────────────────────┐                            │
            │  Optional: Linting         │                            │
            │  files.linting()           │                            │
            └────────────────────────────┘                            │
                         │                                             │
                         ▼                                             │
            ┌────────────────────────────┐                            │
            │  handle_improve_mode()     │                            │
            │  agent.improve()           │                            │
            │  - improve_fn with diffs   │                            │
            │  - validation loop         │                            │
            └────────────────────────────┘                            │
                         │                                             │
                         ▼                                             │
            ┌────────────────────────────┐                            │
            │  compare()                 │                            │
            │  Show colored diff         │                            │
            └────────────────────────────┘                            │
                         │                                             │
                         ▼                                             │
            ┌────────────────────────────┐                            │
            │  User Confirmation         │                            │
            │  "Apply changes?"          │                            │
            └────────────────────────────┘                            │
                         │                                             │
                         └──────────────────┬──────────────────────────┘
                                            │
                                            ▼
                         ┌──────────────────────────────────┐
                         │  stage_uncommitted_to_git()      │
                         │  Stage changes in git            │
                         └──────────────────────────────────┘
                                            │
                                            ▼
                         ┌──────────────────────────────────┐
                         │  files.push(files_dict)          │
                         │  Write files to disk             │
                         └──────────────────────────────────┘
                                            │
                                            ▼
                         ┌──────────────────────────────────┐
                         │  Display Token Usage / Cost      │
                         └──────────────────────────────────┘
```

### Матрица режимов работы

| Флаги | code_gen_fn | process_code_fn | Поведение |
|-------|-------------|-----------------|-----------|
| (none) | gen_code | execute_entrypoint | Стандартная генерация с подтверждением выполнения |
| --clarify | clarified_gen | execute_entrypoint | Интерактивное уточнение → генерация → выполнение |
| --lite | lite_gen | execute_entrypoint | Упрощенная генерация → выполнение |
| --self-heal | gen_code | self_heal | Генерация → автоматические попытки исправления |
| --improve | N/A | N/A | Другой flow через agent.improve() |
| --clarify --self-heal | clarified_gen | self_heal | Уточнение → генерация → автоисправление |
| --lite --self-heal | lite_gen | self_heal | Упрощенная генерация → автоисправление |

### Ключевые компоненты системы

#### 1. CliAgent
Центральный оркестратор, управляющий жизненным циклом:
```python
class CliAgent:
    def init(self, prompt) -> FilesDict:
        # Generation mode
        files_dict = self.code_gen_fn(...)
        entrypoint = gen_entrypoint(...)
        files_dict = self.process_code_fn(...)
        return files_dict
    
    def improve(self, files_dict, prompt) -> FilesDict:
        # Improvement mode
        files_dict = self.improve_fn(...)
        return files_dict
```

#### 2. AI
Абстракция над LLM API:
```python
class AI:
    def start(self, system, user, step_name)
    def next(self, messages, user_input, step_name)
    # Handles token counting, caching, vision support
```

#### 3. Memory (DiskMemory)
Persistent storage:
```python
class DiskMemory:
    def get(self, filename) -> str
    def set(self, filename, content)
    def log(self, log_file, content)
```

Logs:
- `code_gen_log_file.txt` - полный лог генерации
- `improve_log_file.txt` - лог улучшений
- `entrypoint_log_file.txt` - лог создания entrypoint
- `diff_log_file.txt` - ошибки валидации диффов
- `debug_log_file.txt` - отладочная информация

#### 4. ExecutionEnv (DiskExecutionEnv)
Среда выполнения:
```python
class DiskExecutionEnv:
    def upload(self, files_dict)
    def popen(self, command) -> subprocess.Popen
    def run(self, command)
```

#### 5. PrepromptsHolder
Менеджер preprompts:
```python
class PrepromptsHolder:
    def get_preprompts(self) -> Dict[str, str]
    # Loads all preprompts from disk
```

### Поток данных через систему

```
User Input (prompt)
    ↓
PrepromptsHolder (loads preprompts)
    ↓
AI (combines prompts + calls LLM)
    ↓
chat_to_files_dict / parse_diffs (parsing)
    ↓
FilesDict (in-memory representation)
    ↓
Memory.log (logging)
    ↓
ExecutionEnv.upload (write to disk)
    ↓
ExecutionEnv.run (execute)
    ↓
stdout/stderr
    ↓
(if errors and self-heal → back to AI)
```

### Точки расширения

1. **Custom Preprompts**: `--use-custom-preprompts`
2. **Custom Code Gen Function**: Передать свою функцию в CliAgent
3. **Custom Execution Env**: Реализовать BaseExecutionEnv (например, Docker)
4. **Custom Memory**: Реализовать BaseMemory (например, S3)
5. **Custom AI**: Реализовать AI интерфейс (например, для локальных моделей)

---

## ЗАКЛЮЧЕНИЕ

Система gpt-engineer построена на модульной архитектуре с четко разделенными ответственностями:

1. **Пайплайны генерации** (5 режимов):
   - Standard: полная генерация с best practices
   - Clarified: с интерактивным уточнением
   - Lite: упрощенная версия
   - Self-Healing: с автоисправлением
   - Improvement: модификация существующего кода

2. **Композиция промптов**: Гибкая система комбинирования preprompts

3. **Валидация и исправление**: Автоматическая проверка диффов и retry логика

4. **Расширяемость**: Множество точек для кастомизации

**Всего в системе: 5 основных пайплайнов + 1 improvement flow**

