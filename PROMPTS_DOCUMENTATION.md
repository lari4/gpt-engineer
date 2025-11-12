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
