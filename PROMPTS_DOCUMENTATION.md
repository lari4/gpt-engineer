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
