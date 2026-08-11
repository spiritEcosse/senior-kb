## 12. Git

### Основные концепции

**Working tree → Index (staging) → Repository:**

```
working tree  →  git add  →  index (staging)  →  git commit  →  .git/objects
```

```bash
git status                        # что staged, unstaged, untracked
git diff                          # неиндексированные изменения
git diff --staged                 # staged vs последний коммит
git log --oneline --graph --all   # визуальная история веток
```

---

### Ветки и слияние

```bash
git checkout -b feature/my-feature  # создать + переключиться
git merge feature/my-feature        # слить в текущую ветку (создаёт merge commit)
git rebase main                     # переиграть коммиты поверх main (линейная история)
git cherry-pick abc1234             # применить один коммит к текущей ветке
```

**merge vs rebase:**

| | `merge` | `rebase` |
|---|---|---|
| История | Сохраняет — merge commit показывает место соединения | Перезаписывает — линейная, чище |
| Безопасен на общей ветке | ✅ Да | ❌ Нет — перезаписывает общую историю |
| Использовать когда | `main`, `develop`, общие ветки | Локальные feature-ветки перед PR |

!!! warning
    Никогда не rebase ветку, которую используют другие — это перезаписывает SHA коммитов и вынуждает всех делать reset.

---

### Reset, revert, restore

```bash
# git reset — переместить HEAD (и опционально staging/working tree)
git reset --soft HEAD~1    # отменить последний коммит, оставить изменения staged
git reset --mixed HEAD~1   # отменить коммит, оставить unstaged (по умолчанию)
git reset --hard HEAD~1    # отменить коммит, выбросить все изменения ⚠️

# git revert — создать новый коммит, отменяющий предыдущий (безопасно для shared веток)
git revert abc1234

# git restore — отменить изменения в working tree (коммиты не трогает)
git restore file.py           # отменить несохранённые изменения в файле
git restore --staged file.py  # убрать файл из staging
```

---

### Stash

```bash
git stash                       # сохранить изменения, вернуть HEAD
git stash push -m "wip: login"  # с описанием
git stash list                  # список stash-ей
git stash pop                   # применить последний + удалить
git stash apply stash@{2}       # применить конкретный без удаления
git stash drop stash@{0}        # удалить stash
```

---

### Полезные команды инспекции

```bash
git blame file.py              # кто изменил каждую строку и когда
git bisect start               # бинарный поиск коммита с багом
git bisect bad                 # текущий коммит плохой
git bisect good v1.0           # v1.0 был хорошим
# git bisect автоматически чекаутит середину — проверь, затем:
git bisect good / git bisect bad

git reflog                     # история перемещений HEAD — восстановить потерянные коммиты
git shortlog -sn               # количество коммитов по авторам
```

---

### git flow

```
main        — всегда production-ready
develop     — интеграционная ветка
feature/*   — от develop, сливается обратно в develop
release/*   — от develop, сливается в main + develop
hotfix/*    — от main, сливается в main + develop
```

Более простая альтернатива: **GitHub Flow** — ветка от `main`, PR, merge в `main`, деплой.

---

### Исправление ошибок

```bash
# Изменить последний коммит (сообщение или добавить файл)
git add forgotten.py
git commit --amend --no-edit

# Безопасно отменить уже запушенный коммит
git revert HEAD
git push

# Squash последних 3 коммитов в один (интерактивный rebase)
git rebase -i HEAD~3
# В редакторе: первый оставить 'pick', остальные изменить на 'squash'

# Найти удалённый файл
git log --all --full-history -- path/to/file.py
git checkout <sha>^ -- path/to/file.py
```

---

### .gitignore паттерны

```gitignore
*.pyc           # все .pyc файлы
__pycache__/    # директории __pycache__
.env            # файл с секретами
dist/           # результат сборки
!dist/.gitkeep  # кроме этого файла
```

`git rm --cached file` — перестать отслеживать уже закоммиченный файл без удаления локально.

---
