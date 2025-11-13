# GPT-Engineer AI Prompts Documentation

Этот документ содержит полную документацию всех промптов, используемых в приложении gpt-engineer. Промпты сгруппированы по тематикам и функциональному назначению.

---

## 1. БАЗОВЫЕ СИСТЕМНЫЕ ПРОМПТЫ (Base System Prompts)

Эти промпты формируют основу системных инструкций для AI модели и определяют общее поведение генерации кода.

### 1.1. Roadmap Prompt

**Расположение:** `gpt_engineer/preprompts/roadmap`

**Назначение:** Устанавливает базовое ожидание от AI - создать детальную имплементацию с полной архитектурой. Это вводный промпт, который комбинируется с другими промптами для формирования финальной системной инструкции.

**Используется в:**
- `setup_sys_prompt()` - для генерации нового кода
- `setup_sys_prompt_existing_code()` - для улучшения существующего кода

**Промпт:**
```
You will get instructions for code to write.
You will write a very long answer. Make sure that every detail of the architecture is, in the end, implemented as code.
```

---

### 1.2. Generate Prompt

**Расположение:** `gpt_engineer/preprompts/generate`

**Назначение:** Основной промпт для генерации нового кода с нуля. Инструктирует AI думать пошагово, перечислять основные классы/функции, и генерировать полностью функциональный код без placeholder'ов. Включает в себя placeholder `FILE_FORMAT`, который заменяется на содержимое `file_format` промпта.

**Используется в:**
- `setup_sys_prompt()` - основной сценарий генерации кода
- `clarified_gen()` - после фазы уточнения требований

**Особенности:**
- Заменяет `FILE_FORMAT` на содержимое промпта `file_format`
- Требует завершать генерацию фразой "this concludes a fully working implementation"

**Промпт:**
```
Think step by step and reason yourself to the correct decisions to make sure we get it right.
First lay out the names of the core classes, functions, methods that will be necessary, As well as a quick comment on their purpose.

FILE_FORMAT

You will start with the "entrypoint" file, then go to the ones that are imported by that file, and so on.
Please note that the code should be fully functional. No placeholders.

Follow a language and framework appropriate best practice file naming convention.
Make sure that files contain all imports, types etc.  The code should be fully functional. Make sure that code in different files are compatible with each other.
Ensure to implement all code, if you are unsure, write a plausible implementation.
Include module dependency or package manager dependency definition file.
Before you finish, double check that all parts of the architecture is present in the files.

When you are done, write finish with "this concludes a fully working implementation".
```

---

### 1.3. Improve Prompt

**Расположение:** `gpt_engineer/preprompts/improve`

**Назначение:** Промпт для улучшения и модификации существующего кода. Использует формат unified git diff для внесения изменений. Включает в себя placeholder `FILE_FORMAT`, который заменяется на содержимое `file_format_diff` промпта.

**Используется в:**
- `setup_sys_prompt_existing_code()` - для режима улучшения кода
- `improve_fn()` - основная функция улучшения
- `self_heal()` - для автоматического исправления ошибок

**Особенности:**
- Заменяет `FILE_FORMAT` на содержимое промпта `file_format_diff`
- Работает с существующим кодом через git diff формат
- Также требует завершать фразой "this concludes a fully working implementation"

**Промпт:**
```
Think step by step and reason yourself to the correct decisions to make sure we get it right.
Make changes to existing code and implement new code in the unified git diff syntax. When implementing new code, First lay out the names of the core classes, functions, methods that will be necessary, As well as a quick comment on their purpose.

FILE_FORMAT

As far as compatible with the user request, start with the "entrypoint" file, then go to the ones that are imported by that file, and so on.
Please note that the code should be fully functional. No placeholders.

Follow a language and framework appropriate best practice file naming convention.
Make sure that files contain all imports, types etc.  The code should be fully functional. Make sure that code in different files are compatible with each other.
Ensure to implement all code, if you are unsure, write a plausible implementation.
Include module dependency or package manager dependency definition file.
Before you finish, double check that all parts of the architecture is present in the files.

When you are done, write finish with "this concludes a fully working implementation".
```

---

## 2. ПРОМПТЫ ФОРМАТИРОВАНИЯ ВЫВОДА (Output Formatting Prompts)

Эти промпты определяют формат, в котором AI должен выдавать сгенерированный код. Они инжектируются в базовые системные промпты через placeholder `FILE_FORMAT`.

### 2.1. File Format Prompt

**Расположение:** `gpt_engineer/preprompts/file_format`

**Назначение:** Определяет формат вывода для генерации нового кода. Указывает, как AI должен представлять файлы с кодом - в виде имени файла с последующим кодовым блоком.

**Используется в:**
- Инжектируется в `generate` промпт через placeholder `FILE_FORMAT`
- `setup_sys_prompt()` комбинирует его с generate промптом
- `lite_gen()` использует напрямую как системную инструкцию

**Особенности:**
- Простой текстовый формат с именем файла и блоком кода
- Используется для создания новых файлов с нуля
- Минимальные комментарии, фокус на код

**Промпт:**
```
You will output the content of each file necessary to achieve the goal, including ALL code.
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

Do not comment on what every file does. Please note that the code should be fully functional. No placeholders.
```

---

### 2.2. File Format Diff Prompt

**Расположение:** `gpt_engineer/preprompts/file_format_diff`

**Назначение:** Определяет детальный формат unified git diff для внесения изменений в существующий код. Это критически важный промпт с множеством строгих правил для обеспечения точности применения изменений.

**Используется в:**
- Инжектируется в `improve` промпт через placeholder `FILE_FORMAT`
- `setup_sys_prompt_existing_code()` комбинирует его с improve промптом
- Все операции модификации существующего кода

**Особенности:**
- Детальные инструкции по формату unified git diff
- Строгие правила для точного совпадения строк кода
- Специальные правила для создания новых файлов (`--- /dev/null`)
- Предупреждение о том, что программа будет применять диффы автоматически

**Критичные правила:**
- Удаляемые строки (с `-`) и сохраняемые строки (без символа) должны точно соответствовать исходному коду
- Никогда не переносить номера строк из исходных файлов в diff hunks
- Избегать начала hunk с пустой строки
- Все изменения в одном diff chunk на файл

**Промпт:**
```
You will output the content of each file necessary to achieve the goal, including ALL code.
Output requested code changes and new code in the unified "git diff" syntax. Example:

```diff
--- example.txt
+++ example.txt
@@ -6,3 +6,4 @@
    line content A
    line content B
+    new line added
-    original line X
+    modified line X with changes
@@ -26,4 +27,5 @@
        condition check:
-            action for condition A
+            if certain condition is met:
+                alternative action for condition A
        another condition check:
-            action for condition B
+            modified action for condition B
```

Example of a git diff creating a new file:

```diff
--- /dev/null
+++ new_file.txt
@@ -0,0 +1,3 @@
+First example line
+
+Last example line
```

RULES:
-A program will apply the diffs you generate exactly to the code, so diffs must be precise and unambiguous!
-Every diff must be fenced with triple backtick ```.
-The file names at the beginning of a diff, (lines starting with --- and +++) is the relative path to the file before and after the diff.
-LINES TO BE REMOVED (starting with single -) AND LINES TO BE RETAIN (no starting symbol) HAVE TO REPLICATE THE DIFFED HUNK OF THE CODE EXACTLY LINE BY LINE. KEEP THE NUMBER OF RETAIN LINES SMALL IF POSSIBLE.
-EACH LINE IN THE SOURCE FILES STARTS WITH A LINE NUMBER, WHICH IS NOT PART OF THE SOURCE CODE. NEVER TRANSFER THESE LINE NUMBERS TO THE DIFF HUNKS.
-AVOID STARTING A HUNK WITH AN EMPTY LINE.
-ENSURE ALL CHANGES ARE PROVIDED IN A SINGLE DIFF CHUNK PER FILE TO PREVENT MULTIPLE DIFFS ON THE SAME FILE.
```

---

### 2.3. File Format Fix Prompt

**Расположение:** `gpt_engineer/preprompts/file_format_fix`

**Назначение:** Упрощенный промпт для исправления ошибок в ранее сгенерированном коде. Использует тот же формат файлов, что и `file_format`, но с другим контекстом - исправление, а не создание.

**Используется в:**
- Может использоваться в циклах исправления ошибок (legacy, не активно используется в текущей кодовой базе)

**Особенности:**
- Краткий промпт "Please fix any errors in the code above"
- Использует тот же формат вывода, что и file_format
- Фокусируется на исправлении конкретных ошибок

**Промпт:**
```
Please fix any errors in the code above.

You will output the content of each new or changed.
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

Do not comment on what every file does. Please note that the code should be fully functional. No placeholders.
```

---

## 3. ФИЛОСОФИЯ И BEST PRACTICES (Philosophy & Best Practices)

### 3.1. Philosophy Prompt

**Расположение:** `gpt_engineer/preprompts/philosophy`

**Назначение:** Определяет стиль кодирования, best practices и предпочтения для различных языков программирования. Этот промпт добавляется в конец системных инструкций как "Useful to know" и влияет на общее качество генерируемого кода.

**Используется в:**
- `setup_sys_prompt()` - добавляется в конце как "Useful to know:\n" + philosophy
- `setup_sys_prompt_existing_code()` - аналогично для режима улучшения
- Применяется во ВСЕХ режимах генерации и улучшения кода

**Особенности:**
- Универсальные правила для всех языков
- Специфичные рекомендации для Python
- Акцент на структуру проекта и организацию кода
- Требования к файлам зависимостей (requirements.txt, package.json)

**Ключевые принципы:**
- Разные классы в разных файлах
- Комментарии для сложной логики
- Использование pytest и dataclasses для Python
- Следование best practices для структуры файлов

**Промпт:**
```
Almost always put different classes in different files.
Always use the programming language the user asks for.
For Python, you always create an appropriate requirements.txt file.
For NodeJS, you always create an appropriate package.json file.
Always add a comment briefly describing the purpose of the function definition.
Add comments explaining very complex bits of logic.
Always follow the best practices for the requested languages for folder/file structure and how to package the project.


Python toolbelt preferences:
- pytest
- dataclasses
```

---

## 4. ПРОМПТЫ УТОЧНЕНИЯ И ТОЧКИ ВХОДА (Clarification & Entrypoint Prompts)

### 4.1. Clarify Prompt

**Расположение:** `gpt_engineer/preprompts/clarify`

**Назначение:** Управляет фазой уточнения требований перед началом генерации кода. AI задает уточняющие вопросы о неясных аспектах задачи или подтверждает, что всё понятно.

**Используется в:**
- `clarified_gen()` - основная функция с фазой уточнения
- Создает интерактивный диалог с пользователем

**Особенности:**
- Краткий и прямой промпт
- AI должен задать ОДИН уточняющий вопрос или заявить "Nothing to clarify"
- Может делать разумные предположения
- Циклически работает до полного понимания задачи

**Логика работы:**
1. AI получает пользовательский запрос
2. Если что-то неясно - задает ОДИН вопрос
3. Если всё ясно - пишет "Nothing to clarify"
4. Цикл продолжается до явного подтверждения

**Промпт:**
```
Given some instructions, determine if anything needs to be clarified, do not carry them out.
You can make reasonable assumptions, but if you are unsure, ask a single clarification question.
Otherwise state: "Nothing to clarify"
```

---

### 4.2. Entrypoint Prompt

**Расположение:** `gpt_engineer/preprompts/entrypoint`

**Назначение:** Инструктирует AI создать скрипт для запуска сгенерированного кода. Скрипт должен устанавливать зависимости и запускать все необходимые части кода.

**Используется в:**
- `gen_entrypoint()` - генерация исполняемого скрипта
- Создается файл `run.sh` для запуска проекта

**Особенности:**
- Строгое требование: не использовать sudo и глобальные установки
- Только команды, без объяснений
- Использует примерные значения вместо placeholders
- Формат вывода: только code blocks с командами терминала

**Дефолтный пользовательский промпт (если не указан явно):**
```
Make a unix script that
a) installs dependencies
b) runs all necessary parts of the codebase (in parallel if necessary)
```

**Промпт:**
```
You will get information about a codebase that is currently on disk in the current folder.
The user will ask you to write a script that runs the code in a specific way.
You will answer with code blocks that include all the necessary terminal commands.
Do not install globally. Do not use sudo.
Do not explain the code, just give the commands.
Do not use placeholders, use example values (like . for a folder argument) if necessary.
```

---

## 5. ДИНАМИЧЕСКИЕ RUNTIME ПРОМПТЫ (Dynamic Runtime Prompts)

Эти промпты генерируются динамически во время выполнения программы в зависимости от контекста и состояния выполнения.

### 5.1. Self-Healing Error Prompt

**Расположение:** `gpt_engineer/tools/custom_steps.py:111-112`

**Назначение:** Автоматически генерируется когда выполнение сгенерированного кода завершается с ошибкой. Отправляет stdout и stderr обратно в AI для исправления кода.

**Используется в:**
- `self_heal()` функция - автоматическое исправление ошибок выполнения
- Цикл до `MAX_SELF_HEAL_ATTEMPTS` (10 попыток)

**Контекст использования:**
- Запускается после выполнения entrypoint скрипта
- Срабатывает если return code != 0 и != 2
- Использует `improve_fn()` для генерации исправлений

**Шаблон промпта:**
```python
f"A program with this specification was requested:\n{prompt}\n, but running it produced the following output:\n{stdout_full}\n and the following errors:\n{stderr_full}. Please change it so that it fulfills the requirements."
```

---

### 5.2. Diff Refinement Error Prompt

**Расположение:** `gpt_engineer/core/default/steps.py:327-330`

**Назначение:** Генерируется когда AI создал диффы, которые не соответствуют требуемому формату или код не найден в существующих файлах. Запрашивает переписать проблемные диффы.

**Используется в:**
- `_improve_loop()` - цикл улучшения кода с валидацией диффов
- Максимум `MAX_EDIT_REFINEMENT_STEPS` попыток исправления

**Контекст использования:**
- После парсинга и валидации диффов от AI
- Когда `salvage_correct_hunks()` возвращает ошибки
- Повторяется до корректного формата или исчерпания попыток

**Шаблон промпта:**
```python
"Some previously produced diffs were not on the requested format, or the code part was not found in the code. Details:\n"
+ "\n".join(errors)
+ "\n Only rewrite the problematic diffs, making sure that the failing ones are now on the correct format and can be found in the code. Make sure to not repeat past mistakes. \n"
```

---

### 5.3. Clarification Flow Prompts

**Расположение:** `gpt_engineer/tools/custom_steps.py:166-177`

**Назначение:** Набор динамических промптов для управления потоком уточнения требований.

**5.3.1. Assumption Prompt**

Используется когда пользователь пропускает вопрос (нажимает "c" или Enter):

```python
"Make your own assumptions and state them explicitly before starting"
```

**5.3.2. Continuation Prompt**

Добавляется после каждого ответа пользователя для продолжения диалога:

```python
"""
\n\n
Is anything else unclear? If yes, ask another question.\n
Otherwise state: "Nothing to clarify"
"""
```

**Используется в:**
- `clarified_gen()` - интерактивная фаза уточнения

**Логика работы:**
1. AI задает вопрос из clarify промпта
2. Пользователь отвечает или пропускает ("c" или Enter)
3. Если пропустил → AI делает предположения
4. Добавляется continuation prompt для следующего шага
5. Цикл продолжается пока AI не скажет "Nothing to clarify"

---

### 5.4. Lite Generation System Prompt

**Расположение:** `gpt_engineer/tools/custom_steps.py:228`

**Назначение:** Упрощенный режим генерации без roadmap и philosophy, только с форматом файлов.

**Используется в:**
- `lite_gen()` - облегченная версия генерации кода
- Пользовательский промпт используется как system message
- file_format используется как user message

**Особенность:**
```python
messages = ai.start(
    prompt.to_langchain_content(),  # user prompt as system message
    preprompts["file_format"],      # file_format as user message
    step_name=curr_fn()
)
```

Это инвертированная структура по сравнению с обычной генерацией!

---

## 6. КОМПОЗИЦИЯ ПРОМПТОВ (Prompt Composition)

### 6.1. Standard Generation System Prompt

**Формула композиции:**
```
roadmap + generate.replace("FILE_FORMAT", file_format) + "\nUseful to know:\n" + philosophy
```

**Код:** `gpt_engineer/core/default/steps.py:89-94`

**Используется в:**
- Генерация нового кода (`gen_code`)
- После фазы clarification (`clarified_gen`)

---

### 6.2. Code Improvement System Prompt

**Формула композиции:**
```
roadmap + improve.replace("FILE_FORMAT", file_format_diff) + "\nUseful to know:\n" + philosophy
```

**Код:** `gpt_engineer/core/default/steps.py:113-118`

**Используется в:**
- Улучшение существующего кода (`improve_fn`)
- Self-healing (`self_heal`)

---

### 6.3. Message Flow Examples

#### Пример 1: Стандартная генерация

```
[SystemMessage]: roadmap + generate(with file_format) + philosophy
[HumanMessage]: user's prompt
[AIMessage]: generated code with files
```

#### Пример 2: Code Improvement

```
[SystemMessage]: roadmap + improve(with file_format_diff) + philosophy
[HumanMessage]: existing files content
[HumanMessage]: user's improvement request
[AIMessage]: diffs to apply
```

#### Пример 3: Clarified Generation

```
[SystemMessage]: clarify
[HumanMessage]: user's initial prompt
[AIMessage]: clarification question
[HumanMessage]: user's answer + continuation prompt
[AIMessage]: "Nothing to clarify" or next question
... (repeat until clear)
[SystemMessage]: roadmap + generate(with file_format) + philosophy
[HumanMessage]: (all previous clarification messages)
[HumanMessage]: generate prompt
[AIMessage]: generated code
```

---

## 7. SUMMARY: ВСЕ ПРОМПТЫ ПО КАТЕГОРИЯМ

### Статические Preprompts (9 файлов)
1. **roadmap** - базовая установка ожиданий
2. **generate** - генерация нового кода
3. **improve** - улучшение существующего кода
4. **file_format** - формат вывода для новых файлов
5. **file_format_diff** - формат unified git diff
6. **file_format_fix** - формат для исправления ошибок
7. **philosophy** - стиль кодирования и best practices
8. **clarify** - уточнение требований
9. **entrypoint** - генерация скрипта запуска

### Динамические промпты (5 типов)
1. **Self-healing error prompt** - исправление ошибок выполнения
2. **Diff refinement prompt** - исправление некорректных диффов
3. **Assumption prompt** - когда пользователь пропускает вопросы
4. **Continuation prompt** - продолжение диалога clarification
5. **Default entrypoint prompt** - дефолтный запрос для entrypoint

### Композитные промпты (2 основных)
1. **Standard generation** = roadmap + generate(file_format) + philosophy
2. **Code improvement** = roadmap + improve(file_format_diff) + philosophy

---

**Всего уникальных промптов: 16**
- Статических: 9
- Динамических: 5
- Композитных: 2

