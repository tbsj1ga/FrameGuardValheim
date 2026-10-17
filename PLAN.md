# FrameGuard — план реализации

Мод для Valheim, который сам подстраивает графику под нагрузку: когда FPS падает ниже цели
из-за отрисовки — в тяжёлой локации, в большой базе, в бою с эффектами, — он на время снижает
то, что меньше всего заметно, и возвращает обратно, когда запас появился. Если тормозит не
отрисовка, а логика игры (ИИ, физика, сеть), он графику не трогает: картинка хуже, а FPS тот же
— худший исход, которого надо избежать.

Статус: **только план, кода нет.** Документ — для реализации шаг за шагом.

---

## 1. Что уже есть и чем FrameGuard отличается

Проверено по Thunderstore (октябрь 2026).

| Мод | Что делает | Почему этого мало |
|---|---|---|
| LEGIOmods-AutoLodBias 1.1.0 | Двигает только `QualitySettings.lodBias` по FPS: вниз за 0,25 с, при сильной просадке сразу до 1–3. | Не знает причину просадки: при упоре в логику режет прорисовку впустую. Реагирует на рывки подгрузки как на нагрузку. Резкие ступени — дальние объекты «прыгают». |
| Skarif-ValheimPerformanceOverhaul, адаптивное разрешение | Масштаб 3D-картинки по нагрузке GPU. | Нужны GPU timings (`FrameTimingManager`), а в релизной сборке Valheim они выключены (HostProfiler: «Frame timing stats: off»); без них модуль не работает. Часть большого мода. |
| ChangosOF-ValheimOptimizer 2.1.0 | «Dynamic LOD & texture quality». | Последнее обновление 2025-07, до Valheim 1.0. |
| Сама игра | `UpscaledFrameBuffer.UpdateCurrentRenderScale` — «авто»-разрешение. | Считается по DPI экрана, не по FPS. |

Отличия FrameGuard:

1. **Узнаёт узкое место** по шагам кадра движка (PlayerLoop) — без GPU timings.
2. **Отдельная лестница рычагов** под каждое узкое место, от незаметного к заметному.
3. **Проверяет каждый шаг**: рычаг, который не дал выигрыша, возвращается.
4. **Рывки не считаются нагрузкой**; на телепорте, загрузке и при открытой карте — пауза.
5. **Плавные переходы** и асимметрия «вниз быстро, вверх осторожно, с запасом».
6. **Настройки игрока — потолок**, в сохранённые настройки игры ничего не пишется.

---

## 2. Факты движка, на которые опирается план

Проверено по `assembly_valheim.dll` версии 1.0.20 и замерам HostProfiler 0.3.

### 2.1 Шаги кадра (PlayerLoop)

Маркер-делегат между шагами каждой фазы (как в HostProfiler `InstallLoop`) даёт время шага.
Рендер в этой сборке — `LegacyJobified`: отдельного render thread нет, подготовка отрисовки
идёт в главном потоке и видна как шаг.

| Группа | Шаги PlayerLoop | Что значит |
|---|---|---|
| **Logic** | `FixedUpdate.ScriptRunBehaviourFixedUpdate`, `FixedUpdate.PhysicsFixedUpdate`, `Update.ScriptRunBehaviourUpdate`, `Update.ScriptRunDelayedDynamicFrameRate`, `PreLateUpdate.ScriptRunBehaviourLateUpdate`, `PreLateUpdate.DirectorUpdateAnimation*`, `FixedUpdate.Director*` | скрипты игры и модов, физика, анимация |
| **Render CPU** | `PostLateUpdate.FinishFrameRendering`, `PostLateUpdate.UpdateAllRenderers`, `PreLateUpdate.ParticleSystemBeginUpdateAll`, `PostLateUpdate.ParticleSystemEndUpdateAll`, `PostLateUpdate.PlayerUpdateCanvases`, `PostLateUpdate.PlayerEmitCanvasGeometry` | отсечение, тени, команды видеокарте, частицы, UI |
| **Wait** | `TimeUpdate.WaitForLastPresentationAndUpdateTime` (+ `Initialization.*` ожидания, если есть) | ожидание видеокарты/презентации **и** ограничителя FPS |

Списки задаются таблицей имён; неизвестные шаги идут в Other и в решениях не участвуют.

### 2.2 Ожидание: видеокарта или ограничитель

`Wait` включает и намеренный простой ограничителя FPS (`Application.targetFrameRate`).
Различение:

- кадр ≈ интервал цели и Wait большой → цель держится, всё хорошо;
- кадр дольше цели и Wait заметный (> 25% кадра) → **упор в видеокарту**;
- кадр дольше цели, Wait маленький → упор в процессор: Logic или Render CPU — по доле.

### 2.3 Рычаги и как их менять

| Рычаг | API | Что стоит смена | Видимость |
|---|---|---|---|
| Дальность теней от солнца | `QualitySettings.shadowDistance` | ничего | низкая: тени дальше обрываются |
| Дальность прорисовки | `QualitySettings.lodBias` (в игре 1 / 1,5 / 2 / 5) | ничего | средняя: модели вдали проще |
| Разрешение карты теней | `QualitySettings.shadowResolution` | пересоздание shadow map, мелкий рывок | средняя: тени мягче, лесенкой |
| Каскады теней | `QualitySettings.shadowCascades` | мелкий рывок | низкая-средняя: тени вблизи грубее |
| Разрешение 3D-картинки | `UpscaledFrameBuffer.m_targetResolutionVertical` (public static, `uint`; `m_autoTargetResolution = false`) | пересоздание RenderTexture — рывок ~кадр | средняя-высокая: мыло |
| Точечные огни | `QualitySettings.pixelLightCount`, `LightLod.m_lightLimit` / `m_shadowLimit` (public static) | ничего | средняя |

ShadowTrim (если стоит) обрезает тени по слоям как `min(слой, shadowDistance)`, поэтому
снижение `shadowDistance` складывается с ним само, интеграция не нужна.

**Траву (`ClutterSystem`) не трогаем**: смена плотности или дальности перестраивает все патчи
сразу — это рывок, хуже просадки.

### 2.4 Настройки игрока и откаты игры

- `GraphicsSettingsManager.GraphicsSettingsChanged` (static `Action`) — игра применила
  настройки (меню графики, старт). На этом событии FrameGuard: сбрасывает свои отклонения,
  перечитывает **потолок** из `QualitySettings` / `UpscaledFrameBuffer` и начинает заново.
- `GraphicsSettingsManager.ApplyQualitySettings` выставляет lodBias, pixelLightCount,
  shadowDistance/Resolution/Cascades — FrameGuard никогда не вызывает сохранение
  (`SaveGraphicsSettingsToPlayerPrefs`) и не меняет `GraphicsSettingsState`.

### 2.5 Когда не реагировать

| Ситуация | Как узнать |
|---|---|
| телепорт / смерть / заставка | `Player.m_localPlayer.IsTeleporting()`, `IsDead()`, `InIntro()` |
| открыта большая карта / меню / инвентарь | `Minimap.IsOpen()`, `Menu.IsVisible()`, `InventoryGui.IsVisible()` |
| загрузка мира | нет `Player.m_localPlayer`, первые 20 с после спавна |
| сохранение мира | кадр, где идёт `ZNet.SaveWorld` (Harmony prefix/postfix ставит флаг) |
| сборка мусора | `GC.CollectionCount(0)` изменился за кадр |
| окно не в фокусе | `Application.isFocused == false` |

---

## 3. Архитектура

```
FrameGuardPlugin            BepInEx-плагин, конфиг, Update/LateUpdate, консоль
├── FrameClock              маркеры PlayerLoop, время шагов этого кадра
├── FrameStats              кольцевой буфер ~3 с кадров: total/logic/render/wait, флаги
├── Classifier              узкое место: Gpu / RenderCpu / Logic / None
├── Controller              состояние, шаги вниз/вверх, проверка эффекта, cooldown
├── Ladder (x2)             упорядоченные рычаги для Gpu и для RenderCpu
├── Lever (абстракция)      Name, Level 0..N, Ceiling, Apply(level), Transition
│   ├── ShadowDistanceLever
│   ├── LodBiasLever
│   ├── RenderScaleLever
│   ├── ShadowResolutionLever
│   └── PointLightLever
├── Ceiling                 потолок = настройки игрока; событие GraphicsSettingsChanged
├── Pauser                  телепорт, меню, карта, загрузка, сохранение, GC, фокус
└── Overlay (опц.)          одна строка: FPS / цель / узкое место / опущенные рычаги
```

Все файлы — partial-классы одного плагина, как в остальных модах (C# 5, csc из .NET Framework,
`build.ps1` + `check-refs.ps1`). Ссылки: `Unity.Profiling.Core` не нужен, если не берём
ProfilerRecorder; для счётчиков отрисовки (опционально) — нужен.

---

## 4. Алгоритм

### 4.1 Каждый кадр (FrameClock + FrameStats)

1. Маркеры PlayerLoop пишут время шагов; в начале кадра (`Initialization`) — сброс.
2. В конце кадра (последний маркер `PostLateUpdate`) — запись в FrameStats:
   `total = unscaledDeltaTime`, `logic`, `render`, `wait`, флаги `hitch`, `paused`.
3. `hitch` — кадр > 3 × медианы последних 2 с, или был GC, или сохранение, или > 100 мс.
4. Стоимость самого мода — один `Stopwatch.GetTimestamp()` на маркер (~140 маркеров ≈ 5–10 мкс
   на кадр), без аллокаций.

### 4.2 Раз в 0,25 с (Controller.Tick)

Окно — последние 2 с без `hitch` и `paused` кадров (если таких < 60% — тик пропускается).

```
T      = 1000 / targetFps                  // мс
p75    = 75-й процентиль total в окне
logicF = медиана logic / медиана total
renderF= медиана render / медиана total
waitF  = медиана wait / медиана total

if p75 <= T * 1.05:          состояние OK
elif waitF > 0.25:           узкое место GPU
elif renderF >= 0.30 and renderF >= logicF * 0.6:  узкое место RenderCpu
else:                        узкое место Logic
```

Пороги — в конфиге (раздел Advanced).

### 4.3 Снижение (быстро, но не дёргано)

- Условие: не OK, узкое место GPU или RenderCpu, держится **2 с** подряд (8 тиков).
- Берётся **следующий рычаг лестницы** этого узкого места, который ещё не на минимуме и не
  помечен бесполезным.
- Запоминается `before = медиана total` за последнюю 1 с.
- Рычаг опускается **на одну ступень**; запускается переход (см. 4.6).
- Окно ожидания эффекта: переход + **1,5 с** — тики в это время только собирают данные.
- `after = медиана total` за последнюю 1 с окна.
- `gain = before - after`.
  - `gain >= max(0,3 мс, 3% от before)` → шаг засчитан, `lever.GainAt[level] = gain`.
  - иначе → рычаг **возвращается** на ступень назад и помечается бесполезным на **60 с**
    (для этого узкого места); следующий тик пробует следующий рычаг.
- Cooldown после снижения — **1 с**.
- Экстренный режим: p75 > 2 × T в течение 1 с (и это не hitch) → снижать без ожидания эффекта,
  до двух ступеней за тик, только рычагами без рывков (тени, дальность).

### 4.4 Повышение (медленно и с запасом)

- Условие: состояние OK **8 с** подряд.
- Кандидат — **последний опущенный** рычаг (обратный порядок лестницы).
- Запас: `headroom = T - p75`. Поднимать, только если
  `headroom > lever.GainAt[level] × 1,2 + 0,3 мс` — то есть возврат этой ступени,
  по прошлому замеру, точно уложится в цель.
- Если `GainAt` неизвестен (рычаг опускали до перезапуска) — оценка 1,5 мс.
- После повышения: ожидание эффекта 2 с; если цель снова не держится — вернуть рычаг вниз и
  удвоить для него время OK до следующей попытки (до 60 с) — защита от качелей.
- Cooldown после повышения — **3 с**.

### 4.5 Лестницы (по умолчанию)

Лестница RenderCpu (упор в подготовку отрисовки на процессоре):

| # | Рычаг | Ступени (доля потолка или значения) |
|---|---|---|
| 1 | shadowDistance | 100% → 80% → 65% → 50% |
| 2 | lodBias | потолок → −0,5 за ступень, не ниже `MinLodBias` (2,0 — «Высокая» в игре) |
| 3 | pixelLightCount / LightLod limits | потолок → 3/4 → 1/2 |

Лестница GPU (упор в видеокарту):

| # | Рычаг | Ступени |
|---|---|---|
| 1 | shadowDistance | 100% → 80% → 65% → 50% |
| 2 | render scale | 100% → 90% → 80% → 70% (не ниже `MinRenderScale`) |
| 3 | shadowResolution | потолок → −1 ступень → −2 |
| 4 | lodBias | как выше |

Каждый рычаг можно выключить в конфиге; порядок фиксированный.

### 4.6 Плавность

- **shadowDistance, lodBias** — линейный переход за 0,8 с (значение меняется каждый кадр).
- **render scale** — мгновенно (пересоздание RT), не чаще одного изменения в **10 с**, только
  когда узкое место GPU держится стабильно.
- **shadowResolution, cascades** — мгновенно, не чаще раза в 10 с.

### 4.7 Цель FPS

- `TargetFps = 0` (по умолчанию): `Application.targetFrameRate`, если > 0 (ограничитель игры);
  иначе частота монитора `Screen.currentResolution.refreshRateRatio`; иначе 60.
- `TargetFps > 0` — вручную.
- Цель пересчитывается на `GraphicsSettingsChanged` и раз в 5 с.

### 4.8 Выключение и сброс

- Выключение мода (конфиг/консоль/хоткей) — все рычаги мгновенно на потолок.
- Выход из мира, смена сцены — сброс состояния, рычаги на потолок.
- `GraphicsSettingsChanged` — новые потолки, все рычаги на потолок, память GainAt очищается.

---

## 5. Конфиг (`j1ga.frameguard.cfg`)

| Раздел | Ключ | По умолчанию | Что |
|---|---|---|---|
| 01 General | `Enabled` | `true` | |
| 01 General | `TargetFps` | `0` | 0 = ограничитель игры / монитор |
| 01 General | `ShowOverlay` | `false` | строка на экране |
| 02 Levers | `ShadowDistance` | `true` | |
| 02 Levers | `MinShadowDistancePercent` | `50` | |
| 02 Levers | `LodBias` | `true` | |
| 02 Levers | `MinLodBias` | `2.0` | |
| 02 Levers | `RenderScale` | `true` | |
| 02 Levers | `MinRenderScalePercent` | `70` | |
| 02 Levers | `ShadowResolution` | `true` | |
| 02 Levers | `PointLights` | `false` | |
| 03 Advanced | `DownDelay` / `UpDelay` | `2` / `8` с | |
| 03 Advanced | `MinGainMs` | `0.3` | |
| 03 Advanced | `GpuWaitShare` / `RenderShare` | `0.25` / `0.30` | пороги классификатора |
| 03 Advanced | `ToggleKey` | `Alt+F7` | |

## 6. Консоль

| Команда | Что |
|---|---|
| `frameguard` | вкл/выкл |
| `frameguard status` | цель, p75, узкое место и доли logic/render/wait, каждый рычаг: потолок, текущее, GainAt, «бесполезен ещё N с» |
| `frameguard log on/off` | каждое решение в лог: время, причина, рычаг, before/after |
| `frameguard test <рычаг>` | 20 с: рычаг вверх-вниз по 5 с, замер выигрыша (как `shadowtrim bench`) |

---

## 7. Этапы

1. **0.1 — только наблюдение.** FrameClock, FrameStats, Classifier, `status`, оверлей.
   Ничего не меняет. Проверка: на базе, в бою, в Mistlands, после телепорта — правильно ли
   определяется узкое место (сверка с HostProfiler).
2. **0.2 — один рычаг.** Controller + ShadowDistanceLever, обе лестницы из одного рычага.
   Проверка: нет качелей, бесполезный шаг откатывается при упоре в логику.
3. **0.3 — лестницы.** LodBias, RenderScale, ShadowResolution; плавность; Ceiling и
   GraphicsSettingsChanged. Проверка: меню графики во время работы, выход/вход в мир.
4. **0.4 — полировка.** Экстренный режим, `test`, `log`, PointLights, хоткей, README/CHANGELOG
   EN/RU, иконка.
5. **Проверка на слабом ПК** у кого-то из друзей — без неё в описании не обещать цифры.

## 8. Риски и как их закрываем

| Риск | Защита |
|---|---|
| Качели вверх-вниз | асимметрия 2 с / 8 с, запас по GainAt × 1,2, удвоение паузы после неудачного подъёма |
| Режем картинку при упоре в логику | классификатор + проверка эффекта с откатом |
| Рывки при смене разрешения/теней | не чаще раза в 10 с, только при стабильном упоре в GPU |
| Другие моды меняют те же настройки (AutoLodBias, Skarif VPO) | при старте искать их GUID, выключать конфликтующие рычаги и писать в лог |
| Ограничитель FPS путает классификатор | правило 2.2: кадр ≈ цели — значит OK |
| Игра перезаписывает настройки | событие GraphicsSettingsChanged → сброс и новые потолки |
| Стоимость самого мода | ~140 маркеров по одному timestamp, без аллокаций; Tick раз в 0,25 с по буферу ~200 кадров |

## 9. Честные ожидания

- Слабая видеокарта: самый заметный эффект (разрешение, тени), оценочно +20–40% в тяжёлой сцене.
- Слабый процессор, большая база: умеренно (тени, дальность), оценочно +10–20%.
- Бой с толпой мобов: почти ничего — это логика; мод это покажет в `status`, а картинку
  не испортит.
- Рывки (подгрузка, автосохранение) — не его задача.

Цифры — оценки до реализации; в описание мода попадут только замеры.
