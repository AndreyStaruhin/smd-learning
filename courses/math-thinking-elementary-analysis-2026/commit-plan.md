# Commit Plan — v0.2

## Назначение

План переноса новых артефактов в `smd-learning`.

## Предлагаемая структура

```text
courses/math-thinking-elementary-analysis-2026/
  project-systemprompt.md
  course-context.md
  course-mechanics.md
  positions-mathematical-thinking.md
  literature-and-upload-plan.md
  fpf-audit-module-alignment.md
  active-module.md
  heat-map.md
  evidence-ledger.md
  reflection-log.md
  onto-map.md
  commit-plan.md
  modules/
    module-map-v02.md
    module-0-assessment.md
  meta-sessions/
    .gitkeep
  schemes/
    .gitkeep
```

## Коммиты

### 1. Обновить системную инструкцию и контекст

```bash
git add courses/math-thinking-elementary-analysis-2026/project-systemprompt.md \
        courses/math-thinking-elementary-analysis-2026/course-context.md \
        courses/math-thinking-elementary-analysis-2026/course-mechanics.md \
        courses/math-thinking-elementary-analysis-2026/positions-mathematical-thinking.md
git commit -m "course(math): update project instructions for deep foundations v0.2"
```

### 2. Добавить FPF-аудит и карту модулей

```bash
git add courses/math-thinking-elementary-analysis-2026/fpf-audit-module-alignment.md \
        courses/math-thinking-elementary-analysis-2026/modules/module-map-v02.md
git commit -m "course(math): add FPF audit and expanded module map"
```

### 3. Обновить Модуль 0 и активный модуль

```bash
git add courses/math-thinking-elementary-analysis-2026/modules/module-0-assessment.md \
        courses/math-thinking-elementary-analysis-2026/active-module.md
git commit -m "course(math): revise module 0 assessment for deep foundations"
```

### 4. Обновить литературу и рабочие артефакты

```bash
git add courses/math-thinking-elementary-analysis-2026/literature-and-upload-plan.md \
        courses/math-thinking-elementary-analysis-2026/heat-map.md \
        courses/math-thinking-elementary-analysis-2026/evidence-ledger.md \
        courses/math-thinking-elementary-analysis-2026/reflection-log.md \
        courses/math-thinking-elementary-analysis-2026/onto-map.md
git commit -m "course(math): align literature and evidence artifacts with v0.2"
```

## После коммитов

1. Создать или обновить ChatGPT Project.
2. Вставить содержимое `project-systemprompt.md` в Instructions проекта.
3. Подключить репозиторий или загрузить файлы курса.
4. Начать с `modules/module-0-assessment.md`, сессия A0.1.
