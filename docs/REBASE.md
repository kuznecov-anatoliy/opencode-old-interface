# REBASE.md — обслуживание форка opencode-old-interface

Форк: `kuznecov-anatoliy/opencode-old-interface` · upstream: `anomalyco/opencode` · база: `v1.18.30`.
Коммиты поверх базы: c1 free-ping (retry.ts/retry.test.ts), c2 sunset (settings.tsx), c3 фид+guard (electron-builder.config.ts), c4 CI/docs.

## 0. Инварианты
- appId `ai.opencode.desktop`, productName `OpenCode`, userData `%APPDATA%\ai.opencode.desktop`, канал `latest`, `verifyUpdateCodeSignature:false`, `updaterCacheDirName` (`@opencode-aidesktop-updater`), `artifactName` — НИКОГДА не менять.
- LICENSE (MIT) не менять; README-дисклеймер сохранять.
- `packages/desktop/package.json` **отслеживается git'ом**, а поле `version` переписывается на месте скриптом `prepare.ts` при каждой релизной сборке. Не коммитить бамп: после сборки вернуть файл в исходное состояние — `git restore packages/desktop/package.json` (§5.1).
- Версии: мажор релиза форка = мажор **той линии апстрима, которую мы ведём**, **+1**; minor и patch копируются как есть, тег `vX.Y.Z` (§5). Номер **выводится по формуле**, а не выбирается и не сверяется через `max()`, поэтому совпадение с апстримным номером **ведомой линии** исключено по построению. Против **параллельных линий** апстрима (например его `2.x`) эта гарантия не действует — там единственная защита ручная: сверка тега через `git ls-remote --tags upstream` перед созданием тега (§5). Старое правило «наша последняя +1 / max+1» больше не действует; уже выпущенный `v1.18.32` — «до политики». Сейчас ведём линию апстрима `1.x` и потому идём в `2.x`; наш `2.x` **не означает «версию 2.x апстрима»** — у апстрима есть отдельная, параллельная линия `2.x`, которую мы не ведём (§5).
- Локальные коммит и push — **только с `--no-verify`**. Хук `.husky/pre-push` гоняет `bun typecheck` (turbo) и падает на Windows: симлинк `packages/enterprise/src/custom-elements.d.ts` (mode `120000` в индексе) при `core.symlinks=false` разворачивается как обычный текст `../../ui/src/custom-elements.d.ts`, и typecheck спотыкается на нём. Это артефакт чекаута, а не дефект кода; наш CI-workflow typecheck не запускает (`.github/workflows/fork-release.yml` — только install, retry-тест, prepare, build, package, verify). Флаг на коммит — с запасом, на случай появления `pre-commit` с тем же typecheck.
- Пушить только явные теги: `git push --no-verify origin vX.Y.Z`; никогда `git push --tags` (отправит все upstream-теги в форк). `--no-verify` обязателен и для тега — по той же причине, что и для `main` (см. выше): хук `.husky/pre-push` **локальный**, git-клиент на машине, и в CI он не выполняется вовсе, поэтому флаг ничего не отключает в пайплайне.
- Сеть — `Invoke-WithProxy { ... }` (HTTP 127.0.0.1:10809).

## 1. Ребейз на новый upstream-тег
```powershell
$repo = 'C:\Users\Анатолий\Desktop\OpenCode\opencode-old-interface\repo'
Set-Location -LiteralPath $repo
git status --porcelain            # ОБЯЗАТЕЛЬНО пусто: незакоммиченное не переживёт rebase/clean
git rev-parse HEAD                # чекпоинт: записать SHA и держать под рукой до конца ребейза
git config rerere.enabled true    # конфликты запоминаются и переигрываются автоматически
$newTag = 'upstream/v1.19.0'   # подставить актуальный тег
Invoke-WithProxy { git fetch upstream --prune --no-tags }                                  # без тегов — чтобы не перетирать наши релиз-теги
Invoke-WithProxy { git fetch upstream "refs/tags/*:refs/tags/upstream/*" --force --prune } # upstream-теги — в отдельный namespace
$backup = "backup/main-$(Get-Date -Format yyyyMMdd-HHmmss)"
git branch $backup
git switch main
git rebase $newTag              # эквивалент: git rebase --onto $newTag v1.18.30 main
# конфликты — по таблице §2; затем: git add <files>; git rebase --continue (или git rebase --abort)
Invoke-WithProxy { bun install --linker hoisted }   # если менялись зависимости; флаг — windows-паритет с upstream CI
Set-Location packages\opencode; bun test test/session/retry.test.ts; bun typecheck
Set-Location ..\desktop; bun typecheck
Set-Location ..\..
git diff $newTag..main --stat
Invoke-WithProxy { git push --force-with-lease --no-verify origin main }   # --no-verify обязателен: см. §0 (pre-push → bun typecheck падает на Windows-артефакте симлинка)
git branch -D $backup            # после успешного пуша и проверки — backup-ветку удалить
```
После ребейза: **обязательная проверка патча c2** (§1.1), отключить новые upstream-workflow (§4), выбрать версию по политике §5, тег, CI (§5).

### 1.1 Обязательная проверка после ребейза: старый интерфейс не потерян
Отсутствие конфликтов **не** доказывает, что патч жив. Апстрим может удалить логику sunset без конфликта — файл `settings.tsx` существует, но нужных символов в нём уже нет (реальный прецедент: ветка `2.0`, где sunset-логики в файле нет, а патч применился).
```powershell
Select-String -LiteralPath packages\app\src\context\settings.tsx -Pattern 'oldInterfaceSunset','newLayoutDesigns'
# оба маркера должны найтись; если нет — патч c2 фактически потерян, переносить логику заново
```
Дополнительно сверить, что апстрим не ввёл **принудительный** переход на новый layout (жёсткое условие вида `previous === undefined → new layout` вместо флага). Такое изменение ломает форк тихо: код на месте, интерфейс всё равно новый.

### 1.2 База ребейза: по SHA или по namespace, не по имени тега
Одноимённые теги у нас и у апстрима указывают на разные коммиты (`v1.18.32` в форке ≠ `v1.18.32` в `anomalyco/opencode`), поэтому `git rebase v1.18.32` неоднозначен. Фиксировать базу по SHA (`git rebase --onto <sha> <oldsha> main`) либо по тегу из namespace `refs/tags/upstream/*` (как в команде выше). Обычный `git fetch` существующие теги **не** перезаписывает — для перезаписи нужен `--force` / `+refspec` / `--prune-tags`.

## 2. Таблица конфликтов
| Файл | Правило |
|---|---|
| packages/opencode/src/session/retry.ts | Сохранить free-ping: `FREE_LIMIT_WAIT_MS` (60 c), `isFreeLimitError()`, ранний return в `delay()`, снятие cap только для free. При переписанном API — переимплементировать по смыслу |
| packages/opencode/test/session/retry.test.ts | Сохранить тест «free limit waits exactly 60s…»; адаптировать под новый тестовый API |
| packages/app/src/context/settings.tsx | Дата 2099. Если sunset-механизм удалён — решить: переимплементировать доступность старого интерфейса или отбросить патч. Конфликта нет ≠ патч жив: проверять маркеры `oldInterfaceSunset` / `newLayoutDesigns` (§1.1) |
| packages/desktop/electron-builder.config.ts | owner/repo форка + guard `OPENCODE_SIGN`; при смене механизма подписи — guard на новый механизм |
| README.md | Дисклеймер сверху сохранить; остальное — upstream |
| bun.lock / package.json | Как правило upstream; наши патчи зависимости не меняют |
| .github/workflows | Новые upstream-workflow — выключить в форке (§4) |

## 3. Особые случаи
- Upstream удалил старый интерфейс: c2 станет «modify deleted file» → принять удаление и зафиксировать в README, что форк потерял смысл или переименован.
- Upstream удалил только логику sunset, файл оставил: git-конфликта не будет, патч применится «вхолостую» → проверка §1.1.
- Upstream ввёл принудительный переход на новый layout: патч формально жив, интерфейс всё равно новый → переписать условие выбора layout.
- Upstream переписал retry.ts: переписать free-ping по тестам. Если речь о **другой линии `2.x`** апстрима (не о нашей `1.x`, §5) — патч потребует не переноса, а новой разработки. Там `packages/opencode/src/session/retry.ts` переписан, но **механика ретрая от нашей не отличается**: при `FreeUsageLimitError` `retryable()` возвращает непустую строку `GO_UPSELL_MESSAGE` («Free usage exceeded, subscribe to Go https://opencode.ai/go», `retry.ts:60`), а `policy()` прерывает расписание **только** на пустом результате — `if (!message) return Cause.done(...)` (`retry.ts:112`). То есть пустой результат = «прекратить», непустая строка = «показать сообщение пользователю», и **повтор при этом всё равно происходит**; апселл — параллельный UI-эффект, который TUI ловит по событию (`cli/cmd/tui/routes/session/index.tsx:237`), а не блокировка ретрая. **Тот же самый приём уже есть и в нашей линии `1.x`** (`retry.ts:10`, `108-120` — тоже непустой результат = апселл; `policy()` прерывает расписание только на пустом, `retry.ts:201`), поэтому перенос патча на `2.x` автоматически не работает. Различие не в механике, а в том, что в `2.0` **нет наших free-tier патчей вообще**: отсутствуют `FREE_LIMIT_WAIT_MS`, `isFreeLimitError()` и снятие cap по числу попыток (в нашей линии — `retry.ts:204`). Идентичного приёма «остановить повтор» в `2.0` нет. (`QuotaExceeded` здесь ни при чём: в `packages/app/src/utils/persist.ts` это `QuotaExceededError` — переполнение квоты **хранилища браузера**, а не лимит API.)
- Upstream сменил подпись/канал: обновить c3 (guard, owner/repo, channel latest).
- Upstream сменил формат latest.yml: проверить verify-шаг в fork-release.yml.

## 4. Обслуживание workflow в форке (после ребейза)
```powershell
$rep = 'kuznecov-anatoliy/opencode-old-interface'
$wfs = Invoke-WithProxy { gh workflow list -R $rep --all --json id,name,path,state } | ConvertFrom-Json
foreach ($wf in $wfs) {
  if ($wf.path -like '*fork-release.yml') { continue }
  if ($wf.state -ne 'active') { Write-Host "skip $($wf.name) [$($wf.state)]"; continue }
  try { Invoke-WithProxy { gh workflow disable $wf.id -R $rep } } catch { Write-Warning "disable failed for $($wf.name): $($_.Exception.Message)" }
}
Invoke-WithProxy { gh workflow list -R $rep --all }   # глазами: active только fork-release.yml
```

**Уточнение порядка:** сначала включить Actions в настройках репозитория, затем немедленно выполнить disable-loop выше — это закрывает окно гонки, когда могут стартовать лишние workflow. Важно: НЕ отключать Actions глобально (enabled=false) — это заблокирует и наш workflow.

**Инвариант pull_request_target:** при ребейзе (§1) проверять, что новые upstream-workflow не используют `pull_request_target` с опасными секретами (например, `secrets.GITHUB_TOKEN` в контексте PR из форка). Если обнаружен — отключить или ограничить `permissions`.

## 5. Политика версий и релиз
Номер релиза форка = версия **той линии апстрима, которую мы ведём (сейчас `1.x`)**, с увеличенной MAJOR-цифрой; minor и patch копируются без изменений:
`1.18.33 → 2.18.33`, `1.18.34 → 2.18.34`.

Почему мажор всегда ровно на единицу больше: номер **выводится** из апстримного по формуле `MAJOR+1.MINOR.PATCH`, а не выбирается вручную и не сверяется через `max()`. Прибавление единицы к мажору необратимо, поэтому совпадение с апстримным номером **ведомой линии** исключено **по построению** — сверка с ведомой линией не нужна. Против **параллельных линий** апстрима (`2.x`) гарантия не распространяется: там защита только ручная — обязательная сверка тега через `git ls-remote --tags upstream` перед созданием тега (команда ниже). Плюс номер в «О программе» сразу отличает форк от официальной сборки.

**Обязательная оговорка: у апстрима ДВЕ линии.** Кроме ведомой нами `1.x` апстрим параллельно развивает **отдельную линию `2.x`** — `v2.0.0 … v2.0.23` и далее (проверяется: `git ls-remote --tags upstream`, `gh api repos/anomalyco/opencode/releases`). Это другая переработка, старого интерфейса там нет. Мы её **не ведём** и наши патчи туда не переносим (перенос = новая разработка, §3). Следствие: наш `2.x` означает лишь «мы следуем за линией `1.x`, где старый интерфейс ещё жив»; читать наш `2.x` как «версию 2.x апстрима» нельзя. Мажор форка не прибит к `2.x` навечно — он следует за мажором ведомой линии апстрима, и если у неё сменится мажор, наш уйдёт в следующий (`3.x`).

Старое правило «наша последняя + 1 / max + 1» **больше не действует**. Уже выпущенный `v1.18.32` — «до политики». Тег `vX.Y.Z`, как раньше — только явный пуш тега (§0).

```powershell
$major = 2                                                   # мажор нашей линии (= мажор ведомой линии апстрима + 1)
git tag --list "v$major.*"                                  # что уже выпущено нами в этой линии
git tag --list                                               # все наши теги
Invoke-WithProxy { gh release list -R anomalyco/opencode --limit 5 --exclude-pre-releases }  # свежесть upstream
Invoke-WithProxy { git ls-remote --tags upstream "refs/tags/v$major.*" }   # ОБЯЗАТЕЛЬНО: нашего тега не должно быть у апстрима — ручная защита от коллизии с параллельными линиями (см. выше)
# version = апстрим-версия ведомой линии с MAJOR+1  →  v2.18.33
git tag -a v2.18.33 -m "OpenCode Old Interface v2.18.33"
Invoke-WithProxy { git push --no-verify origin v2.18.33 }     # --no-verify: см. §0; → CI fork-release.yml
```

## 5.1 Локальная сборка (обязательный порядок)

Предпосылки — без них шаги ниже не работают:
- `bun install` **в корне репозитория**: `prepare.ts` импортирует `@opencode-ai/script`, а `build`/`package` берут `electron-vite` и `electron-builder` из воркспейса.
- Версия bun должна удовлетворять `packageManager` из корневого `package.json` (`bun@1.3.14`). Проверку делает `packages/script/src/index.ts:16-18`: при несовпадении он бросает `This script requires bun@^1.3.14, but you are using bun@<ваша>`. То же дублирует `.husky/pre-push`.

Порядок: номер версии записывается в `packages/desktop/package.json` скриптом `packages/desktop/scripts/prepare.ts` из переменной `OPENCODE_VERSION`, и этот шаг обязан выполниться **ДО** `build`.

```powershell
$repo = 'C:\Users\Анатолий\Desktop\OpenCode\opencode-old-interface\repo'
Set-Location -LiteralPath $repo
bun install                              # 0. предпосылка: воркспейс и версия bun (§ выше)
Set-Location -LiteralPath "$repo\packages\desktop"
$env:OPENCODE_CHANNEL = 'prod'        # обязателен: без него fallback даёт dev и appId ai.opencode.desktop.dev
$env:OPENCODE_VERSION = '2.18.33'     # по политике §5
bun ./scripts/prepare.ts              # 1. записать версию — ДО build
bun run build
bun run package:win -- --publish never
```
`OPENCODE_CHANNEL=prod` — не опция: уже был инцидент с appId `ai.opencode.desktop.dev` в собранном инсталляторе.
Отдельная строка `bun run prebuild` **избыточна**: `packages/desktop/scripts/prepare.ts:4` уже выполняет `await import("./prebuild")`.

Почему порядок важен: версия рендера **запекается на этапе build**. `packages/desktop/src/renderer/index.tsx` читает `version: pkg.version` из `package.json`, и в собранном чанке остаётся литерал вида `const version = "1.18.30"` — но **не в исходнике**, а в артефакте сборки `packages/desktop/out/renderer/assets/main-*.js` (имя файла содержит хеш и меняется от сборки к сборке).

Если `prepare` пропустить **целиком**, расхождения не будет: и electron-builder, и рендер читают один и тот же `packages/desktop/package.json` — просто везде останется старая версия (и релиз получится не тот, что задумано). Реальный сценарий расхождения — **`build` выполнен раньше `prepare`** либо `out/renderer` не пересобран: в бандле тогда старая версия, а в нативной части инсталлятора — новая (electron-builder берёт версию из `package.json` в момент `package:win`). Никаких конфликтов в git при этом нет.

Проверка после сборки — все три значения должны совпасть с тегом:
```powershell
$ver = $env:OPENCODE_VERSION
# 1. версия, запечённая в рендер-бандл (хеш в имени файла меняется — берём по glob)
Select-String -Path "$repo\packages\desktop\out\renderer\assets\*.js" -Pattern ('const version\s*=\s*"' + [regex]::Escape($ver) + '"')
# 2. версия в package.json — её же возьмёт electron-builder
(Get-Content "$repo\packages\desktop\package.json" | ConvertFrom-Json).version
# 3. ProductVersion собранного инсталлятора
[System.Diagnostics.FileVersionInfo]::GetVersionInfo("$repo\packages\desktop\dist\opencode-desktop-win-x64.exe").ProductVersion
# вернуть файл в исходное состояние, чтобы бамп не попал в коммит (§0)
git restore packages/desktop/package.json
```
Проверка (1) — ровно та же, что теперь делает CI в шаге Verify (`fork-release.yml`). Логи и маркеры патчей в `app.asar` — §8.

## 6. Откат релиза
```powershell
# мягкий: пометить prerelease (клиенты с allowPrerelease=false его не увидят)
Invoke-WithProxy { gh release edit vX.Y.Z -R $rep --prerelease }
# жёсткий: удалить релиз и тег — только если релизом ещё никто не обновился!
Invoke-WithProxy { gh release delete vX.Y.Z -R $rep --yes --cleanup-tag }
git tag -d vX.Y.Z
```
Правило: если релиз уже установлен клиентами — не удалять, выпустить исправленный `+1`.

## 7. Кэш апдейтера (pending)
- `%LOCALAPPDATA%\@opencode-aidesktop-updater\pending\` — скачанный инсталлятор (в т.ч. от официального фида, например официальный 1.18.31).
- Перед ручной сменой фида/откатом: закрыть приложение через UI и удалить каталог:
```powershell
Get-ChildItem -LiteralPath "$env:LOCALAPPDATA\@opencode-aidesktop-updater\pending" -ErrorAction SilentlyContinue
Remove-Item  -LiteralPath "$env:LOCALAPPDATA\@opencode-aidesktop-updater\pending" -Recurse -Force
```
- Записи `opencode.updater` в `%APPDATA%\ai.opencode.desktop` обновляются сами; отдельно чистить не нужно.
- Это кэш приложения, не системная настройка.

## 8. Диагностика
- Логи: `%APPDATA%\ai.opencode.desktop\logs\<stamp>\main.log` (строки `auto updater configured`, `updater state changed`).
- Фид: `%LOCALAPPDATA%\Programs\@opencode-aidesktop\resources\app-update.yml` (owner/repo = наши).
- Маркеры патчей в `app.asar`: `FREE_LIMIT_WAIT_MS`, `oldInterfaceSunset`, `Date(2099, 0, 1)`.
- CI: `gh run list -R $fork -L 5`; `gh release view vX.Y.Z -R $fork`.
