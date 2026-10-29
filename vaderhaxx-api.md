# vaderhaxx — Lua API

Документация публичного API читающегося из `vaderhaxx.lua` (≈20 550 строк). Все ссылки на строки даны
по состоянию на момент написания — ищите по имени функции, а не только по номеру.

---

## 1. Как получить доступ

Скрипт сам публикует себя в `getgenv()`:

```lua
getgenv().vaderhaxx          -- таблица scriptLib (см. ниже)
getgenv().vaderhaxx.loaded   -- false до полной загрузки, true после
```

Флаг `loaded` выставляется в самом конце (`vaderhaxx.lua:20541`), после инициализации всех модулей и UI:

```lua
repeat task.wait(0.1) until getgenv().vaderhaxx and getgenv().vaderhaxx.loaded
local net = getgenv().vaderhaxx.modules.cheat.networking
net.send("modcmd", "/votekick:someone:cheats")   -- реальный пример из самого чита (vaderhaxx.lua:9222)
```

> **Важно:** всё тело чита исполняется как строка — `run([=[ ... ]=])` (`vaderhaxx.lua:51`), то есть
> через `loadstring`, либо через `run_on_actor(mainActor, ...)` если найден актор `ClientLoader`.
> Поэтому внутренние `local`-переменные снаружи **не видны**;对外 доступна только таблица
> `getgenv().vaderhaxx` и глобальная `ui`.

### Дерево экспорта (`scriptLib`, `vaderhaxx.lua:20056`)

```text
getgenv().vaderhaxx
├── loaded                     boolean
├── modules
│   ├── game                   pfModules — модули самого Phantom Forces
│   └── cheat
│       ├── playerInfo         трекер игроков
│       ├── currentInfo        текущее оружие/контроллер
│       ├── hooks              низкоуровневая подмена функций
│       ├── tickbase           управление тиками
│       ├── networking         отправка/перехват сетевых запросов
│       ├── heap               бинарная куча (newHeap)
│       ├── pathfinding        A*/BFS
│       ├── mathematics        математика и траектории
│       ├── rayCaster          рейкасты
│       ├── spring             пружины
│       ├── signal             собственный signal
│       ├── gunHandler         бинды на статы оружия
│       └── encryption         AES (ECB/CBC/CFB/OFB/CTR)
├── cheatFunctions
│   ├── legit | rage | esp | visuals | misc | sharedHooks
├── menu
│   ├── mainMenu               = ui (экземпляр меню)
│   ├── keybindMenu            = keybindsui
│   ├── utilities              утилиты UI (signal, копирование массивов и т.д.)
│   └── Drawings               все созданные Drawing-объекты
└── ui                         = uilibrary (класс/фабрика, метод :start)
```

---

## 2. `modules.game` — pfModules

Доступ к игровым модулям PF (через `getgenv().vaderhaxx.modules.game`). Используются самим читом:

| Поле | Назначение |
| --- | --- |
| `NetworkClient` | `NetworkClient:send(request, ...)` — нативная отправка (без хуков чита) |
| `CharacterInterface` | `getCharacterObject()`, `_rootPart` — локальный персонаж |
| `CameraInterface` | `getActiveCamera("MainCamera")` |
| `GameClock` | `getTime()` — игровое время (для бэктрека) |
| `PlayerDataClientInterface` | `getPlayerData()` — уровень/статы аккаунта |
| `VoteKickInterface` | `vote("yes"/"no"/"dismiss"/"priority"/...)` |
| `FirearmObject` | `fireRound`, `canFire`, `impulseSprings` (перехвачены хуками) |
| `MeleeObject` | `canMelee` (перехвачен) |
| `CharacterObject` | `jump`, `step` (перехвачены) |
| `ReplicationObject` | репликация третьего лица |

---

## 3. `modules.cheat.networking`

```lua
networking.send(request: string, ...)                    -- отправка мимо хуков? нет: идёт через trampoline,
                                                          -- но добавленные addHook для него не применяются повторно
networking.addHook(request, priority, callback)          -- перехват ИСХОДЯЩЕГО запроса (client -> server)
networking.addListener(request, priority, callback)      -- приём ВХОДЯЩЕГО пакета (server -> client)
```

Семантика (реализация `vaderhaxx.lua:10528-10634`):

* оба списка сортируются по `priority` **по убыванию** — больший приоритет исполняется первым;
* колбэк получает один аргумент — **таблицу `args`** (массив);
* **`return true` из `addHook`** — пакет блокируется и не уходит на сервер;
* **`return true` из `addListener`** — цепочка слушателей этого события прерывается;
* `addHook` вешается на `NetworkClient.send` через `hooks.trampoline`, `addListener` — на `RemoteEvent.OnClientEvent`.

### Известные события

| Событие | Направление | Где используется |
| --- | --- | --- |
| `repupdate` | hook | позиция/углы, tickbase |
| `newbullets` | hook + listener | выстрелы, авто-пик, чамсы попаданий |
| `bullethit` | hook | подавление отдачи/звуков, рейдж-хиты |
| `equip`, `reload`, `swapweapon`, `getammo` | hook | оружие |
| `aim`, `stance`, `sprint` | hook | stance/анти-аим |
| `spawn`, `ping`, `falldamage` | hook/listener | спавн, тики, урон |
| `newgrenade` | hook + listener | телепорт гранат, предупреждения |
| `sendChatMessage` | hook | килл-сей, чат-спам |
| `killfeed` | listener | ивент-логи, киллсей, vote kick on death |
| `smallaward`, `updatestats` | listener | серии (streak), счётчики |
| `correctposition` | listener | руббербэнд сервера |
| `startvotekick` | listener | автоматика vote kick |
| `bulkplayerupdate` | listener | ESP/статусы игроков |

Пример:

```lua
local V = getgenv().vaderhaxx
V.modules.cheat.networking.addListener("killfeed", 1, function(args)
    print(args[1].Name, "->", args[2].Name)
end)
```

---

## 4. `modules.cheat.tickbase`

```lua
tickbase:getTickBase()        -- текущее значение тика для исходящего пакета
tickbase:getMinimumTickBase() -- min(текущий, minTick)
tickbase:shift(time: number)  -- сдвинуть базу на time секунд, возвращает новое значение тика
tickbase.tickShift            -- число накопленных сдвигов
tickbase.minTick              -- минимум, навязанный исходящим аргументам
tickbase.outGoingTickArgs     -- { request = argIndex } — какие запросы несут тик
```

Типичное применение: `tickbase:shift(delta)` после ускорения скорострела (`vaderhaxx.lua:19730`).

---

## 5. `modules.cheat.pathfinding`

```lua
pathfinding.canTraverse(origin, direction)  -- true, если отрезок проходим (используется в auto peek)
pathfinding.optimizePath(path, step)        -- упрощение ломаной
pathfinding.lerpPos(start, goal, step)
pathfinding.visualizePath(waypoints, color) -- отладочная визуализация
pathfinding.bestFirstSearch(origin, target, param)
pathfinding.floorBestFirstSearch(origin, target, param)
pathfinding.vadAStar(param)                 -- основной A* вадерхакса
pathfinding.floorAStar(param) / pathfinding.intAStar(param) / pathfinding.aStar(param)
```

`param` — таблица с `origin`, `target`, `processTime` (бюджет на кадр), размером ноды и т.п.
Алгоритм выбирается в UI: `rage_pathfindingtype` → `options.pathfindingAlgorithim`
(ключи `"a*"` / `"bfs"`), узлы/бюджет: `rage_pathfindingnodesize`, `rage_pathfindingtime`.

---

## 6. `modules.cheat.mathematics`

```lua
-- углы и направления
mathematics.normalizeAngle(angle)
mathematics.pitchYawToLookVec(pitch, yaw)
mathematics.lookVecToPitchYaw(lookVec)
mathematics.angleBetweenVector3(originCf, vec2)
mathematics.angleBetweenVector2(v1, v2)
mathematics.rotationMatrix(vec, angle)
mathematics.safeUnit(vec, epsilon)

-- экран
mathematics.worldToViewportPoint(worldPosition, clampEdge, clampOffset) -- -> Vector2, onScreen
mathematics.worldToScreenPoint(pos)
mathematics.spherePoint(worldPosition, radius)

-- траектории (гранаты / телепорт снаряжения)
mathematics.solveQuadratic(a0, a1, a2)
mathematics.solveTrajectory(initialPosition, gravityVector, targetPosition, bulletSpeed)
mathematics.trajectory(origin, acceleration, target, speed, extra)
mathematics.raySphereIntersection(origin, direction, center, radius)

-- прочее
mathematics.truncateNumber(number, decimalPlaces)
mathematics.map(x, start0, stop0, start1, stop1)
```

---

## 7. `modules.cheat.playerInfo` и `currentInfo`

```lua
playerInfo.list                  -- [Player] = tracker
playerInfo.init()                -- подписки на репликацию/спавн/смерть (вызывается читом сам)
playerInfo.track(plr)            -- добавить игрока в трекинг
playerInfo.remove(plr)           -- убрать
```

Поля трекера: `plr`, `position`, `angles`, `velocity`, `health`, `alive`, `stance`, `aiming`,
`sprint`, `character` (хеш модели → ключи хитбоксов), `updates` (история позиций с `time`
/ `receivedTime` — то, что ест бэктрек), сигналы `spawned` / `died`, `misses`, `resolving`.

```lua
currentInfo.heldWeapon()     -- объект активного оружия (или nil)
currentInfo.getWeaponInfo()  -- активный WeaponController
```

---

## 8. `modules.cheat.hooks`, `rayCaster`, `heap`, `signal`, `spring`, `gunHandler`, `encryption`

```lua
hooks.replace(old, new)                       -- обычная подмена, возвращает обёртку с .old
hooks.trampoline(source, key, new)            -- подмена метода source[key]; new(self, ...) вызывает .old
hooks.list                                    -- реестр

rayCaster.rayCast(origin, direction, ignoreList, ignoreFunc, keepIgnoreListChanges, ignoreWater)
rayCaster.raycastSingleExit(origin, direction, part)
rayCaster.bulletIgnored(instance)             -- фильтр для пуль
rayCaster.physicsIgnore                        -- стандартный ignore-list

heap.newHeap(comp)                            -- бинарная куча с компаратором

signal = utilities.signal
Signal.new() / :Connect(handler) / :Fire(...) / :Wait() / :Destroy()

spring = Spring                                -- пружина (амортизация), используется visuals/thirdPerson

gunHandler.createBind()                        -- бинда на запрос статов оружия
gunHandler.mapsToFunctions(stat, mappings)
gunHandler.createStatHook(fnName)
gunHandler.keysFrom(data) / gunHandler.defaultMap(...)

encryption.encrypt_ECB(key, plainText) / encryption.decrypt_ECB(key, cipherText)
encryption.encrypt_CBC / decrypt_CBC  (+ CFB/OFB/CTR в той же таблице)   -- bytes -> bytes
```

---

## 9. `cheatFunctions`

### 9.1 `rage`

Главные точка входа — `rage.think(options, playerStatus, heldWeapon, deltaTime)` (вызывается каждый
кадр, `vaderhaxx.lua:14834`). Внутри:

```lua
rage.getEnemies() / rage.filterEnemies(enemies, maxFov, origin, originDirection)
rage.scan(base, enemyBase, origins, points, weaponData, options, additionalData)
rage.getScanOrigins(base, enemyBase, options)     -- откуда стрелять
rage.getScanPoints(enemyBase, options)            -- куда стрелять (многоточечный сдвиг)
rage.getScanDirections(mode, amount, facing)      -- режимы: cardinal/random/circle/corner/snake/
                                                   -- sphere/fibonacci/hemisphere/layered
rage.pickHitbox(choices, character)               -- первая отмеченная часть тела, есть у цели
rage.processResult(base, results, enemy, weaponData, options)
rage.processAimBot(base, localVelocity, enemySet, weaponData, options)
rage.resolvePosition(records, additionalData)     -- резолвер
rage.autoWall(origin, launching_velocity, acceleration, landing_time, maximum_penetration, step_size)
rage.canTeleport() / rage.traverseTeleports(movements, inversed, cancelflybypass)
rage.firePositionScanning(origin, target, maxdist)
rage.teleportGrenade(args, base, enemyPool, grenadeData)
rage.knifeAura(base, enemies, options) / rage.processKnifeBot(...)
rage.currentTarget   -- {player, pInfo, part, ...} текущая цель
rage.baseFirePos / rage.baseAngles / rage.baseAngles2
rage.fakePosition    -- позиция анти-аима (работает при anti aim)
rage.teleporting / rage.teleportfailed / rage.baseAntiAimAngles / rage.jitterAngles
```

Ключевые `options` (собираются в `vaderhaxx.lua:14768-14829`):

```lua
enabled, maximumFov, knifeEnable, knifeRadius, knifeThroughWalls, knifePathfinded, waitForEquip,
autowallHitScan, autowallHitScanIncrement, teleportingThreshold, maximumHitscanningPoints,
autowallHitscanDistance, sorting (= rage_hitscanselection), sortingBias, preference (= rage_sorting),
pathfinding, pathfindingAlgorithim, pathfindEnemyPosition, pathfindFailedCardinal,
maximumWallBang, shiftHitbox, shiftHitboxIncrements, maximumHitboxShift,
cardinalShift/randomShift/shiftSphere/shiftFibonacci/shiftHemisphere/shiftLayered,
cardinalDirections/randomDirections/circleDirections/cornerDirections/snakeDirections,
sphereDirections/fibonacciDirections/hemisphereDirections/layeredDirections,
maximumBackTrack (сек), backtrackSamples, backtrackSorting, resolver, autoFire, predictDamage,
silentAim, hitboxes (мультивыбор), hitbox (активный), priorityOnly,
skyboxBullets, skyboxHeight, autoPeek, peekRadius, peekScans, peekGrounded, peekFill,
bulletSpeed, autoWallTime, waitForSpawn, canTeleport
```

### 9.2 `legit`

```lua
legit.getEnemies() / legit.filterEnemies(enemies, maxFov, minFov, origin, originDirection)
legit.prioritizeEnemies(enemies, origin, originDirection, targetting)
legit.getScanPoints(enemy, hitboxes, origin, originDirection, accuracy, priority)
legit.scan(scanPoints, origin, autowall)
legit.moveMouse(delta)                -- плавный донос мыши
legit.getBarrel()                     -- позиция дула
legit.processAimAssist(deltaTime)     -- аим-ассист
legit.processSilentAim(bulletArgs)    -- silent aim
legit.processTriggerBot(deltaTime)    -- триггербот
legit.hitGroups                       -- {"Head", "Torso", "Left Arm", ...} ключи хитбоксов
legit.currentTarget / legit.bulletBinds
```

### 9.3 `esp`

```lua
esp.createDrawing(type, prop)      -- Drawing.new + применение свойств, регистрируется в Drawings
esp.initializeDrawingObjects(onScreen, offScreen, general)
esp.getBoundingBox(headCf, rootCf, llegCf, rlegCf, cacheTable)
esp.apply(player) / esp.remove(player)
esp.espData                        -- [Player] = данные отрисовки
esp.defaultProperties              -- дефолтные пропсы боксов/скелетов/текстов
esp.mainTexts / esp.flagTexts / esp.boneLines / esp.gradentHealthBarSegments
esp.physicalFolder                 -- Folder в CoreGui для физических объектов
esp.dropped                        -- выпавшее оружие
```

### 9.4 `visuals`

```lua
visuals.thirdPerson                -- состояние камеры третьего лица (анимации repupdate и т.д.)
visuals.updateWorldColor() / visuals.updateWorldColorLoop / visuals.skyboxes / visuals.updatesky()
visuals.saveViewmodel() / visuals.reverseViewmodel() / visuals.updateViewmodelchams()
visuals.savePlayerchams() / visuals.reversePlayerchams() / visuals.updatePlayerchams()
visuals.bulletTracer(origin, target, color, transparency, thickness)
visuals.hitChams(plr, color, a, t)
visuals.grenadeWarning / visuals.grenadeLines / visuals.onClientNewGrenade
visuals.materials / visuals.textures / visuals.basepartProperties
visuals.fovCircles / visuals.fakeIndicator
```

### 9.5 `misc`

```lua
misc.log(event, text)      -- ивент-лог, фильтруется по ui.flags.misc_eventlogs[event],
                            -- троттлинг 0.5 c, уведомление на misc_eventlogtime секунд
misc.checkStaff()          -- проверка всех по группе StyLiS (1103278), при находке — hopServer
misc.hopServer()           -- переход на другой сервер, возвращает true/false
misc.autoVote(player, statusTable, options)  -- голосование по playerListStatus (priority/friendly/neutral)
misc.autodeploy            -- соединение: авто-деплой при отсутствии персонажа
misc.speedHack / misc.flyHack / misc.noClip / misc.floorTeleports / misc.bypassFly
misc.getSpawnPlates() / misc.getNearestSpawnPlates(start, spawnPlates)
misc.killsay / misc.chatSpammer / misc.killsays / misc.streakSounds
misc.playSound(id, level) / misc.autoJump(humanoid, keys) / misc.onFallDamage(args)
misc.laserPointer           -- лазерная указка (Drawing-объекты внутри)
misc.lastRubberBand         -- tick() последнего руббербэнда
```

Доступные события для `misc.log` (значения дропдауна `misc_eventlogs`):
`kill`, `death`, `bullet hit`, `ragebot peek`, `server correction`, `vote kick`, `streak`.

### 9.6 `sharedHooks`

```lua
sharedHooks.firearmObjectcanFireHook
sharedHooks.firearmObjectimpulseSpringsHook
sharedHooks.meleeObjectcanMeleeHook
sharedHooks.characterObjectStepHook
-- каждый — обёртка hooks.trampoline, реальный вызов лежит в .old
```

---

## 10. UI API (`ui` / `menu`)

Экземпляр: `getgenv().vaderhaxx.menu.mainMenu`. Все креаторы вызываются через двоеточие и
принимают **одну таблицу параметров**.

### 10.1 Креаторы элементов

| Метод | Основные параметры | Поля `.value` |
| --- | --- | --- |
| `ui:createtoggle(p)` | `tab`, `subsection`, `name`, `flag`, `value`, `tooltip`, `detected` | `boolean` |
| `ui:createslider(p)` | `tab`, `subsection`, `name`, `flag`, `value`, `minimum`, `maximum`, `suffix`, `custom`, `tooltip` | `number` |
| `ui:createdropdown(p)` | `tab`, `subsection`, `name`, `flag`, `values = {{name, default}, ...}`, `multichoice`, `tooltip` | `table[name] -> boolean` |
| `ui:createkeybind(p)` | `tab`, `subsection`, `object` (родитель!), `name`, `flag`, `parentflag` | `boolean` (клавиша зажата/активна) |
| `ui:createcolorpicker(p)` | `tab`, `subsection`, `object`, `name`, `flag`, `color` | `.color` (Color3), `.transparency` |
| `ui:createbutton(p)` | `tab`, `subsection`, `name`, `flag`, `confirmation` | `.pressed` (Signal) |
| `ui:createtextbox(p)` | `tab`, `subsection`, `text`, `flag` | `string` |
| `ui:createnotification(p)` | `text`, `lifetime` (сек), `priority` (0 инфо / 1 предупр. / 2 критично) | — |

Разное:

* `detected = true` — элемент рисуется оранжевым (`255,106,79`), смысл: «может детектиться».
* `parentflag` — у keybind: привязка к тоглу, который он активирует.
* `values` у дропдауна — массив `{{имя, включено_по_умолчанию}, ...}`; даже у `multichoice = false`
  `.value` остаётся таблицей `имя -> bool`.

### 10.2 Объект флага

```lua
ui.flags.<flag>
    .type     -- "toggle" | "slider" | "dropdown" | "keybind" | "color" | "button" | "textbox"
    .name     -- отображаемое имя
    .value    -- см. таблицу выше
    .changed  -- Signal: :Connect(function(newValue) ... end)
    .element  -- ссылка на элемент UI (у toggle/slider)
    .parentflag, .key, .activation, .tab, .section, .object  -- у keybind
```

Чтение:

```lua
local ui = getgenv().vaderhaxx.menu.mainMenu
if ui.flags.rage_enabled.value then ... end
ui.flags.rage_enabledkey.changed:Connect(function(on) print("rage", on) end)
```

### 10.3 Меню и состояния

```lua
ui:openui() / ui:closeui()        -- открыть/закрыть меню (клавиши insert / delete / backquote)
ui:savestate()                    -- сериализовать все флаги
ui:loadstate(state)               -- восстановить
ui:getconfigs()                   -- список сохранённых конфигов
ui:updateaccent() / ui:updatecolors() / ui:updatemenuanimations()
ui.accent                         -- Color3 текущего акцента
ui.uiopen                         -- меню открыто
ui:startWatermark                 -- включить ватермарку
ui:calltooltip(text, object, offset) / ui:callcolorpicker(name, flag, position, transparency)
ui:getnextidentifier()
```

### 10.4 Инвентарь вкладок

`menu.directory[tab][subsection]`, элементы — `menu.elements[tab][subsection][name]`:

| tab | subsection (кол-во элементов) |
| --- | --- |
| `legit` | aim assist (22), bullet redirection (10), trigger bot (9) |
| `rage` | aimbot (17), hack vs. hack (24), anti aimbot (18), misc (11) |
| `esp` | enemy (43), team (34), dropped (8), esp settings (13) |
| `visuals` | local (17), camera (13), viewmodel (20), world (36) |
| `misc` | movement (22), weapon modifications (9), extra (31) |
| `config` | ui (11), extra (5), other (6) |
| `players` | options (3) |

---

## 11. Примеры

**Свой обработчик киллфид + уведомление**

```lua
local V = getgenv().vaderhaxx
local ui = V.menu.mainMenu

V.modules.cheat.networking.addListener("killfeed", 5, function(args)
    ui:createnotification({text = args[1].Name .. " fragged " .. args[2].Name, lifetime = 4, priority = 0})
    return true -- перекрывает стандартный обработчик чита
end)
```

**Своя опция в меню**

```lua
local ui = getgenv().vaderhaxx.menu.mainMenu
ui:createtoggle({tab = "misc", subsection = "extra", name = "my toggle", flag = "misc_mything",
    value = false, tooltip = "example"})
ui.flags.misc_mything.changed:Connect(function(v) print("my toggle ->", v) end)
```

**Прицел на ближайшую голову через rage-пайплайн**

```lua
local V = getgenv().vaderhaxx
local rage = V.cheatFunctions.rage
local enemies = rage.getEnemies()
print(#enemies)
```

**Отправка сетевой команды в обход UI**

```lua
getgenv().vaderhaxx.modules.cheat.networking.send("repupdate", position, angles, angles2, tick)
```

---

## 12. Ограничения и заметки

1. **Только то, что в `getgenv().vaderhaxx`** — локальные переменные внутри тела чита недоступны
   из-за `run([=[ ... ]=])`. Если нужного модуля нет в `scriptLib`, его надо туда дописать
   (блок `-- script lib`, `vaderhaxx.lua:20055`).
2. **`ui.flags` глобален** внутри среды исполнения чита, но снаружи надёжнее брать
   `V.menu.mainMenu.flags`.
3. **`detected`-элементы** (staff detector, skybox bullets, resolver-опции) помечены автором как
   рискованные — трогайте осознанно.
4. **Приоритеты хуков** — всегда указывайте `priority` явно; слушатели идут по убыванию.
5. **`return true` блокирует пакет** — из `addHook` это отправка «в никуда», из `addListener` —
   прерывание цепочки. Злоупотребление ломает ESP/резолвер.
6. Идентификаторы флагов должны быть уникальны: `menu.flags[flag]` перезаписывается при
   дубликате, старый элемент перестаёт работать.
