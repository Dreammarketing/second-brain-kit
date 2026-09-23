# Етап 9 — Все автоматично

Після цього етапу памʼять працює сама. Ти просто працюєш у будь-якій папці
і закриваєш сесію, коли закінчив. Усе інше відбувається без тебе:

- **розмови** — iai-pme пише сам
- **база знань** — сесія закривається сама: лог, картка проєкту, фокус тижня
- **памʼять і граф** — отримують той самий запис одночасно з базою
- **сесія вилетіла** — закривається сама на наступному старті
- **граф вимкнений** — запис чекає в черзі і відправляється, коли граф оживе

`/brain close` руками лишається для важливих сесій — він точніший, бо питає тебе.
Для звичайних не треба нічого.

**Час:** 30-40 хвилин. Робиться один раз.

---

## ПРОМПТ — копіювати звідси до кінця файлу

Це фінальний етап Second Brain Kit — зробити так, щоб уся памʼять
працювала автоматично. Етапи 0-8 я вже пройшов.

Поводься так само: по кроках, коротко, «Ідемо далі?» у кінці кожного.
Секрети не показуй і не пиши у файли, що йдуть у Git.
Наявні налаштування і хуки не затирай — тільки додавай.

## Крок 0. Подивись, що в мене стоїть

Сам, без питань до мене, перевір і покажи однією таблицею:

- де лежить моя база знань (шукай папку з `.claude/skills/brain/SKILL.md`)
- де лежать мої робочі проєкти (спитай, якщо не очевидно)
- iai-pme: `iai-mcp doctor`, `iai-mcp daemon status`, `iai-mcp capture-hooks status`
- Graphiti: чи є в `claude mcp list`, чи запущений контейнер (`docker ps`)
- що вже є в `~/.claude/settings.json` у розділі `hooks`

## Крок 1. Автозахоплення розмов

Переконайся, що iai-pme пише розмови **сам**:
`iai-mcp capture-hooks status` має показати, що хуки встановлені.

Якщо ні — `iai-mcp capture-hooks install`, потім перевір ще раз.

Поясни мені одним реченням, що тепер записується без моєї участі.

## Крок 2. `/brain` працює з будь-якої папки

Зараз `/brain` видно тільки, коли я відкриваю сесію в базі знань.
А працюю я в папках проєктів. Виправ:

1. Зроби `/brain` глобальним — символьне посилання
   `~/.claude/skills/brain` → `<моя база>/.claude/skills/brain`.
   Посилання, а не копія: правлю в одному місці — працює скрізь
2. Додай мою базу в `~/.claude/settings.json` →
   `permissions.additionalDirectories`, щоб з будь-якої папки
   Claude міг писати в базу без дозволу щоразу

## Крок 3. Шляхи — щоб Claude знав, де що

У `~/.claude/CLAUDE.md` (глобальний, читається в кожній сесії) допиши блок:

```
## Мої шляхи
- База знань (Obsidian): <шлях до бази>
- Робочі проєкти: <шлях до проєктів>/<назва-проєкту>/
- Нові робочі файли, креативи, транскрибації — тільки в папку проєкту
- Рішення і підсумки — в базу; сесії закриваються автоматично
- Нову папку проєкту створюю тільки після мого «так»
- Якщо я пишу «перевір автозакриття» — покажи останні записи
  ~/.claude/logs/brain-autoclose.log і останні записи «(автозакриття)» в log.md
```

Підстав мої реальні шляхи з кроку 0. Нічого не переносимо.

## Крок 4. `/brain close` розносить усе сам

Відкрий `<база>/.claude/skills/brain/SKILL.md`, розділ `close`.
Після наявних кроків (лог, CONTEXT.md, INDEX.md, now.md) додай:

```
5. Памʼять асистента: той самий запис логу передай в iai-pme
   інструментом memory_capture — з назвою проєкту на початку.
6. Граф — тільки якщо Graphiti підключений у claude mcp list:
   передай запис логу інструментом add_memory (у старих версіях
   add_episode), group_id = назва проєкту.
   Якщо Graphiti не відповідає — не падай. Допиши запис у
   <база>/.graph-queue.md з датою і назвою проєкту.
7. Перед кроком 6 перевір .graph-queue.md: якщо там є записи
   і граф живий — спершу відправ їх, потім очисти файл.
8. Одним рядком скажи, куди що пішло: база ✓ · памʼять ✓ · граф ✓/черга/немає
```

`.graph-queue.md` додай у `.gitignore` бази.

Покажи мені, що змінилось у SKILL.md, до того як зберегти.

## Крок 5. Автозакриття сесій

Тепер прибираємо останній ручний крок. Два хуки Claude Code:

- **коли сесія закінчується** — у фоні запускається Claude і сам робить
  `/brain close` за транскриптом. Запис позначається «(автозакриття)»
- **коли сесія починається** — перевіряється, чи немає минулих сесій,
  які вилетіли і не закрились. Якщо є — закриваються у фоні

Що НЕ закривається автоматично, і це правильно:
- порожні сесії (менше 4 реплік і жодної правки файлів)
- сесії, де я вже зробив `/brain close` руками
- сесії, відкриті зараз в іншому вікні

Поясни мені двома реченнями, як це працює і що автозакриття пише
без мого підтвердження — тому важливі сесії краще закривати руками.

**1. Скрипт.** Створи `~/.claude/hooks/brain_session.py` рівно з таким
вмістом, тільки заміни `__BRAIN_BASE__` на повний шлях до моєї бази:

```python
#!/usr/bin/env python3
"""Автозакриття /brain для Second Brain Kit.

  start — реєструє сесію; ті, що вилетіли без закриття, закриває у фоні
  end   — закриває поточну сесію у фоні, якщо в ній була робота

Підключається хуками SessionStart → start і SessionEnd → end у ~/.claude/settings.json.
"""
import json
import os
import shutil
import subprocess
import sys
import time
from pathlib import Path

BASE = Path("__BRAIN_BASE__").expanduser()  # шлях до бази знань, підставляється при встановленні
MIN_TURNS = 4        # менше реплік і жодної правки файлів — сесія порожня, не пишемо
IDLE_HOURS = 3       # незакрита сесія без змін довше — вважаємо, що вилетіла
RETRY_HOURS = 1      # фонове закриття ще може йти — не запускаємо вдруге

STATE = Path.home() / ".claude" / "brain-sessions"
LOG = Path.home() / ".claude" / "logs" / "brain-autoclose.log"
EDIT_TOOLS = {"Edit", "Write", "MultiEdit", "NotebookEdit"}
ALLOWED = ("Read,Glob,Grep,Edit,Write,Skill,mcp__iai-mcp__memory_capture,"
           "mcp__graphiti__add_memory,mcp__graphiti__add_episode")


def inspect(transcript):
    """Повертає (була робота, вже закрито руками)."""
    turns, edited, closed = 0, False, False
    try:
        f = open(transcript, encoding="utf-8")
    except (OSError, TypeError):
        return False, False
    with f:
        for line in f:
            try:
                entry = json.loads(line)
            except ValueError:
                continue
            content = (entry.get("message") or {}).get("content")
            is_user = entry.get("type") == "user"
            if is_user and isinstance(content, str):
                turns += 1
                if "<command-name>/brain</command-name>" in content and "close" in content:
                    closed = True  # руками набрали /brain close
            if not isinstance(content, list):
                continue
            for part in content:
                if not isinstance(part, dict):
                    continue
                if is_user and part.get("type") == "text":
                    turns += 1
                if part.get("type") == "tool_use":
                    name, args = part.get("name"), part.get("input") or {}
                    if name in EDIT_TOOLS:
                        edited = True
                    if name == "Skill" and args.get("skill") == "brain" and "close" in str(args.get("args", "")):
                        closed = True
    return (turns >= MIN_TURNS or edited), closed


def close_async(sid, transcript, cwd, mark):
    """Запускає фоновий Claude, який робить /brain close. Маркер знімається тільки після успіху."""
    LOG.parent.mkdir(parents=True, exist_ok=True)
    log = open(LOG, "a", encoding="utf-8")
    log.write(f"\n=== {time.strftime('%Y-%m-%d %H:%M')} · {sid} · {cwd}\n")
    log.flush()
    claude = shutil.which("claude")
    if not claude:
        log.write("claude не знайдено в PATH — пропускаю\n")
        log.close()
        return False
    prompt = (
        f"Автозакриття сесії. Прочитай транскрипт {transcript} (робоча папка сесії: {cwd}). "
        "Виконай /brain close для проєкту, над яким там працювали, БЕЗ питань — мене немає поруч. "
        "Сам визнач, що зроблено, що вирішено, що лишилось відкритим. "
        "Кожен запис у log.md познач «(автозакриття)». Не дублюй того, що вже є в log.md. "
        "Якщо реальної роботи не було — нічого не пиши. "
        "Якщо не можеш визначити проєкт — поклади підсумок в inbox бази."
    )
    mark.write_text(json.dumps({"transcript": transcript, "cwd": cwd, "launched": time.time()}))
    env = dict(os.environ, BRAIN_AUTOCLOSE="1", BRAIN_PROMPT=prompt, BRAIN_MARK=str(mark), BRAIN_CLAUDE=claude)
    subprocess.Popen(
        ["sh", "-c", f'"$BRAIN_CLAUDE" -p "$BRAIN_PROMPT" --allowedTools "{ALLOWED}" && rm -f "$BRAIN_MARK"'],
        cwd=str(BASE), env=env, stdin=subprocess.DEVNULL, stdout=log, stderr=log, start_new_session=True,
    )
    log.close()
    return True


def on_start(data):
    STATE.mkdir(parents=True, exist_ok=True)
    sid = data.get("session_id", "")
    started = 0
    for mark in STATE.glob("*.json"):
        if mark.stem == sid:
            continue
        try:
            info = json.loads(mark.read_text())
        except (OSError, ValueError):
            mark.unlink(missing_ok=True)
            continue
        if time.time() - info.get("launched", 0) < RETRY_HOURS * 3600:
            continue  # закриття вже йде у фоні
        tr = info.get("transcript", "")
        changed = os.path.getmtime(tr) if os.path.exists(tr) else mark.stat().st_mtime
        if time.time() - changed < IDLE_HOURS * 3600:
            continue  # можливо, ще відкрита в іншому вікні
        worked, closed = inspect(tr)
        if worked and not closed and close_async(mark.stem, tr, info.get("cwd", ""), mark):
            started += 1
        else:
            mark.unlink(missing_ok=True)
    if sid and not (STATE / f"{sid}.json").exists():
        (STATE / f"{sid}.json").write_text(
            json.dumps({"transcript": data.get("transcript_path", ""), "cwd": data.get("cwd", "")}))
    if started:
        print(f"[brain] Незакритих минулих сесій: {started}. Закриваю автоматично у фоні — "
              "запис зʼявиться в log.md з позначкою «(автозакриття)».")


def on_end(data):
    sid = data.get("session_id", "")
    mark = STATE / f"{sid}.json"
    tr = data.get("transcript_path", "")
    worked, closed = inspect(tr)
    if worked and not closed:
        STATE.mkdir(parents=True, exist_ok=True)
        close_async(sid, tr, data.get("cwd", ""), mark)
    else:
        mark.unlink(missing_ok=True)


def main():
    if os.environ.get("BRAIN_AUTOCLOSE"):
        return  # фоновий Claude не закриває сам себе
    try:
        data = json.load(sys.stdin)
    except ValueError:
        data = {}
    try:
        {"start": on_start, "end": on_end}[sys.argv[1]](data)
    except Exception as e:  # хук ніколи не має ламати сесію
        LOG.parent.mkdir(parents=True, exist_ok=True)
        with open(LOG, "a", encoding="utf-8") as f:
            f.write(f"помилка хука ({' '.join(sys.argv[1:])}): {e!r}\n")


if __name__ == "__main__":
    main()
```

**2. Хуки.** У `~/.claude/settings.json`, розділ `hooks`, **додай** до наявних
(там уже є хуки iai-pme — їх не чіпай):

```json
"SessionStart": [{"hooks": [{"type": "command", "command": "python3 \"$HOME/.claude/hooks/brain_session.py\" start"}]}],
"SessionEnd":   [{"hooks": [{"type": "command", "command": "python3 \"$HOME/.claude/hooks/brain_session.py\" end"}]}]
```

Якщо `SessionStart` чи `SessionEnd` там уже є — додай новий елемент
у той самий масив, не заміняй. Покажи мені підсумковий розділ `hooks`
до збереження і перевір, що JSON валідний.

**3. Сухий тест.** Прожени скрипт на тестовому транскрипті, не запускаючи
справжнє закриття: подай на вхід JSON із вигаданою сесією через змінну
`BRAIN_AUTOCLOSE=1` і переконайся, що скрипт тихо виходить без помилок.
Потім покажи, що `~/.claude/brain-sessions/` створюється.

## Крок 6. Граф — лишаємо чи прибираємо

Пропусти цей крок, якщо Graphiti в мене немає.

Якщо є — спитай мене один раз: «Граф лишаємо чи прибираємо?»
І скажи чесно: iai-pme уже пише розмови і має свій граф звʼязків.
Graphiti додає тільки точну історію фактів у часі, а коштує Docker,
~10 ГБ диска, ~2 ГБ памʼяті і гроші OpenAI.

**Якщо лишаємо** — зроби безпечним і живучим:

1. У `docker-compose.yml` перед кожним портом `127.0.0.1:` —
   щоб граф не був відкритий у Wi-Fi
2. `restart: unless-stopped` — щоб піднімався сам після перезапуску
3. `ENCRYPTION_KEY` (64 hex) у `.env`, якщо вебінтерфейс на :3000 не пускає
4. Модель у `config.yaml` — велика. Маленькі ламають витяг, це відома
   поведінка Graphiti
5. Нагадай мені виставити місячний ліміт витрат на ключ OpenAI
6. Покажи, як зупиняти без втрат: спершу `redis-cli SAVE`, потім stop
7. Перезапусти контейнер і перевір, що граф живий і дані на місці

**Якщо прибираємо:**

1. Спершу переконайся, що все з графа є в базі — граф похідний,
   його завжди можна перебудувати з бази
2. `docker compose down`, видали образ, прибери graphiti з `claude mcp list`
3. Покажи, скільки місця звільнилось
4. Docker Desktop видаляти тільки після мого «так»

## Крок 7. Наскрізна перевірка

Прожени при мені:

1. `/brain close` руками з цієї сесії — перевір три місця:
   - у базі — запис у `log.md` проєкту зʼявився
   - в iai-pme — знайди цей запис пошуком по памʼяті
   - у графі (якщо лишили) — знайди цей факт пошуком
2. Якщо граф є — зупини Docker, зроби ще один `/brain close`,
   переконайся, що запис ліг у `.graph-queue.md`. Запусти Docker,
   `/brain close` ще раз — черга має відправитись і очиститись

Про кожен пункт — чесно ✓ або ✗. Що не пройшло — лагодь одразу.

Потім скажи мені дослівно:

> Останній тест — автозакриття. Відкрий нову сесію в папці будь-якого
> проєкту, зроби там дрібну річ на 4-5 повідомлень і закрий сесію.
> Через хвилину відкрий ще одну і напиши: «перевір автозакриття».

## Крок 8. Підсумок

Одним коротким повідомленням:

- що тепер відбувається **само**, без мене
- коли варто закривати руками (важливі сесії з рішеннями)
- де лежать дані кожної памʼяті і як зробити бекап
- де лог автозакриття: `~/.claude/logs/brain-autoclose.log`
- як вимкнути автозакриття, якщо заважає: прибрати два хуки з `settings.json`
