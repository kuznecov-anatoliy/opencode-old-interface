# REBASE.md — обслуживание форка opencode-old-interface

Форк: `kuznecov-anatoliy/opencode-old-interface` · upstream: `anomalyco/opencode` · база: `v1.18.30`.
Коммиты поверх базы: c1 free-ping (retry.ts/retry.test.ts), c2 sunset (settings.tsx), c3 фид+guard (electron-builder.config.ts), c4 CI/docs.

## 0. Инварианты
- appId `ai.opencode.desktop`, productName `OpenCode`, userData `%APPDATA%\ai.opencode.desktop`, канал `latest`, `verifyUpdateCodeSignature:false`, `updaterCacheDirName` (`@opencode-aidesktop-updater`), `artifactName` — НИКОГДА не менять.
- LICENSE (MIT) не менять; README-дисклеймер сохранять.
- `packages/desktop/package.json` (version) генерируется `prepare.ts` — НИКОГДА не коммитить (локально `git restore`).
- Версии: **MAJOR+1 к апстриму** (§5), plain semver, тег `vX.Y.Z`. Старое правило «наша последняя +1 / max+1» больше не действует; уже выпущенный `v1.18.32` — «до политики», с `2.18.33` и далее только `2.x`.
- Пушить только явные теги: `git push origin vX.Y.Z`; никогда `git push --tags` (отправит все upstream-теги в форк).
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
Invoke-WithProxy { git push --force-with-lease origin main }   # pre-push hook: bun typecheck (turbo) — идёт долго
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
- Upstream переписал retry.ts: переписать free-ping по тестам. Если речь о будущей линии 2.x апстрима — патч потребует не переноса, а новой разработки: там переписан `retry.ts`, а `QuotaExceeded` считается неретраибельным.
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

**Уточнение порядка:** сначала включить Actions (§2.5), затем немедленно выполнить disable-loop (§2.6) — это закрывает окно гонки, когда могут стартовать лишние workflow. Важно: НЕ отключать Actions глобально (enabled=false) — это заблокирует и наш workflow.

**Инвариант pull_request_target:** при ребейзе (§1) проверять, что новые upstream-workflow не используют `pull_request_target` с опасными секретами (например, `secrets.GITHUB_TOKEN` в контексте PR из форка). Если обнаружен — отключить или ограничить `permissions`.

## 5. Политика версий и релиз
Номер релиза форка = версия апстрима с увеличенной MAJOR-цифрой:
`1.18.33 → 2.18.33`, `1.18.34 → 2.18.34`; если апстрим дойдёт до `2.x` → `3.x`.

Причина: апстрим всегда в `1.x`, форк всегда в `2.x` — коллизия номеров исключена в принципе, без сверки и `max()`. Плюс номер в «О программе» сразу отличает форк от официальной сборки.

Старое правило «наша последняя + 1 / max + 1» **больше не действует**. Уже выпущенный `v1.18.32` — «до политики»; с `2.18.33` и далее только `2.x`. Тег `vX.Y.Z`, как раньше — только явный пуш тега (§0).

```powershell
git tag --list "v2.18.*"                                   # что уже выпущено нами
Invoke-WithProxy { gh release list -R anomalyco/opencode --limit 5 --exclude-pre-releases }  # свежесть upstream
# version = апстрим-версия с MAJOR+1  →  v2.18.33
git tag -a v2.18.33 -m "OpenCode Old Interface v2.18.33"
Invoke-WithProxy { git push origin v2.18.33 }               # → CI fork-release.yml
```

## 5.1 Локальная сборка (обязательный порядок)
Номер версии записывается в `packages/desktop/package.json` скриптом `packages/desktop/scripts/prepare.ts` из переменной `OPENCODE_VERSION`. Этот шаг обязан выполняться **ДО** `build`.

Почему: версия рендера запекается на этапе build — в бандле `packages/desktop/src/renderer/index.tsx` остаётся `const version = "1.18.30"` (значение из `package.json`, §0: файл не коммитится). Если `prepare` пропущен, electron-builder возьмёт версию из `package.json` — получится расхождение: нативная версия правильная, а в приложении показана старая. Никаких конфликтов в git при этом нет.

```powershell
$repo = 'C:\Users\Анатолий\Desktop\OpenCode\opencode-old-interface\repo'
Set-Location -LiteralPath "$repo\packages\desktop"
$env:OPENCODE_CHANNEL = 'prod'        # обязателен: без него fallback даёт dev и appId ai.opencode.desktop.dev
$env:OPENCODE_VERSION = '2.18.33'     # по политике §5
bun ./scripts/prepare.ts              # 1. записать версию — ДО build
bun run prebuild
bun run build
bun run package:win -- --publish never
```
`OPENCODE_CHANNEL=prod` — не опция: уже был инцидент с appId `ai.opencode.desktop.dev` в собранном инсталляторе.

Расхождение renderer/native диагностируется так: §8, маркеры в `app.asar` + «О программе» / `ProductVersion` у exe.

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
