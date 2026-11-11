# VR Revolver Range: Unity 6 + XR Interaction Toolkit 3.x

Реалистичный VR-тир: револьвер на поясе, ручной взвод курка, откидной барабан, перезарядка по одному патрону. Никаких кнопок перезарядки и подсказок. Цель: Quest 3 (OpenXR, URP), 90 Гц.

> Код написан без доступа к Unity и не компилировался. Статическая проверка прошла (скобки, имена файлов и классов, перекрёстные вызовы), но первый запуск покажет мелкие расхождения с твоей версией XRI. Список мест для сверки в конце файла.

## 1. Архитектура

Принцип: каждая деталь знает только своё состояние и сообщает о нём событиями. `Revolver` единственный оркестратор.

```
Рука (HandRef) ──grip/trigger/палец──► HammerGrip / HammerThumb ─► Hammer ─(OnIndex)─► Cylinder.Advance
                                      PushSlider(защёлка) ──────► Cylinder.SetLatch
                                      CylinderHandle ───────────► Cylinder (качание, вращение)
Trigger (главная рука) ─► RevolverTrigger ─(OnBreak)─► Revolver ─► Hammer.Fall ─(12 мс)─► Strike
Strike: камора напротив бойка → Cartridge.Fire → Bullet (BulletPool) → IShootable (Target) → ScoreSystem
Cartridge в руке ─► Chamber.OnTriggerStay (допуски) ─► SeatIn (60 мс) ─► Cylinder открыт/закрыт
```

| Папка | Скрипты |
|---|---|
| Weapon | `Revolver`, `Hammer`, `RevolverTrigger`, `Cylinder`, `Chamber`, `Cartridge`, `Holster`, `TwoHandAim`, `Bullet`, `BulletPool`, `RevolverAudio`, `Haptics` |
| Interaction | `HandRef`, `HandZone`, `HammerGrip`, `HammerThumb`, `PushSlider`, `CylinderHandle` |
| Range | `Target`, `IShootable`, `ScoreSystem`, `AmmoTable` |
| Player | `ComfortSettings`, `ComfortLocomotion`, `VignetteRing`, `HeightCalibration` |

Решения, которые стоит знать:
- **Поза оружия ведётся скриптом** (`TwoHandAim`), а не XRI: у `XRGrabInteractable` выключены Track Position/Rotation. Одна рука: чистый трекинг без задержки. Две руки: опорная кисть тянет `SupportPoint`, вес 0.7, сглаживание ≈17 мс. В кобуре позу задаёт attach-точка сокета.
- **Журавль барабана двигается скриптом**, а не `HingeJoint`: вложенный Rigidbody под движущимся родителем в VR дрожит.
- **Свободная рука работает через зоны** (триггер-коллайдеры + `HandRef`), а не через XRI-хват. Поэтому grip на барабане не превращается в обычный хват оружия.

## 2. Механика в числах

| Часть | Значения |
|---|---|
| Курок | 0° покой; 49° сухарь цепляет; 52° боевой взвод; 4 щелчка каждые 13°; барабан +60° за 40 мс на 36° хода; пружина возврата 600°/с |
| Спуск | ход 14°; срыв на 80%; сброс на 40%; без взвода ход ≤ 3°, выстрела нет |
| Выстрел | падение курка 12 мс; вспышка 40 мс; отдача 8° и 20 мм за 35 мс, возврат 220 мс; двумя руками ×0.55 |
| Барабан | защёлка 8 мм, порог 80%, только при опущенном курке; журавль 0–95°; фиксатор при ≤ 3°; шток 25 мм, порог 90% |
| Выброс | живые вываливаются при стволе вверх ≥ 45°; стреляные только штоком или тряской > 8 м/с²; 0.35 м/с назад; по одному раз в 80 мс |
| Вставка патрона | угол ≤ 25°, смещение ≤ 7 мм, донце у среза; посадка 60 мс |
| Патрон | 15 г; капсула R 5 мм, L 39 мм; зона захвата R 2 см; Velocity Tracking, Damping 0.7 |
| Револьвер | 1.1 кг; хват grip ≥ 0.70, отпуск ≤ 0.40 |
| Пуля | 300 м/с; падение ≈14 см на 50 м; пул 32 |
| Движение | 1.4 м/с, разгон 6 м/с²; рывок 0.25 с на 6 м/с; snap-turn 30°; виньетка 110°→70°, вход 0.15 с, выход 0.35 с |

Хаптика: выстрел 1.0/45 мс + хвост 0.35/90 мс (опорная рука ×0.6); трещотка 0.15/12 мс; сухарь 0.25/15 мс; защёлка 0.4/20 мс; патрон в камере 0.3/20 мс; сухой выстрел 0.35/20 мс.

## 3. Сборка сцены

Пакеты: XR Interaction Toolkit 3.x, OpenXR, Input System, TextMeshPro, URP. Tracking Origin Mode = Floor.

**Слои:** Hand, Weapon, Cartridge, Casing, Target, Env. В Physics отключить: Cartridge×Weapon (патрон проходит сквозь корпус, вставку решает `Chamber`), Casing×Hand. Маска пули: Target + Env. `Time.fixedDeltaTime = 1/90`, Solver Iterations 8.

**Револьвер (префаб):**
```
Revolver        Rigidbody(1.1 кг), XRGrabInteractable, Revolver, TwoHandAim, RevolverAudio
├─ GripPoint    attach transform; ось Z вдоль ствола
├─ SupportPoint ~0.07 м вперёд от рукояти, чуть ниже ствола
├─ StrikePoint  на оси ствола у заднего среза барабана
├─ Muzzle
├─ GripColliders  рукоять и рамка. ТОЛЬКО они в XRGrabInteractable.colliders
└─ RecoilPivot  родитель всей визуальной части (не GripPoint)
   └─ Frame
      ├─ HammerPivot (Hammer: ось X, покой identity) ─ HammerZone (SphereCollider trigger R 0.025, HammerGrip)
      ├─ TriggerMesh (RevolverTrigger)
      ├─ LatchSlider (PushSlider kind=Latch, travel 0.008, BoxCollider trigger)
      ├─ HammerThumb (на корне, стики из Input Actions)
      └─ SwingPivot  под барабаном, ось Z ‖ стволу (Cylinder ссылается на него)
         ├─ Rotor ─ Chamber×6 (Chamber + SphereCollider trigger R 0.012, дочерний Mouth: +Z вглубь каморы)
         ├─ CylinderZone (SphereCollider trigger R 0.05, CylinderHandle)
         └─ EjectorRod (PushSlider kind=Ejector, travel 0.025, axis = ось штока)
```
`XRGrabInteractable`: Select Mode Multiple, Throw On Detach off, Attach Transform = GripPoint (остальное `Revolver` выставляет сам).

**Патрон (префаб):** Rigidbody, XRGrabInteractable (Velocity Tracking), `Cartridge`; капсула R 5 мм (`bodyColliders`), отдельная сфера-триггер R 2 см в `colliders` интерактабла; слой Cartridge.

**Руки:** контроллер = XRDirectInteractor + `HandRef` + `HapticImpulsePlayer`; коллайдеры (kinematic, слой Hand): сфера R 0.04 на ладони, капсула на указательном. `pinchPoint` и `indexTip` задать вручную.

**Кобура:** объект `Holster` + XRSocketInteractor (Show Interactable Hover Meshes = off, радиус 0.05, Starting Selected Interactable = револьвер).

**Тир:** `ScoreSystem` + табло TMP, `BulletPool`, `AmmoTable` (стол и слой Cartridge), мишени на 7/15/25 м (`Target`, слой Target, центр = `center`), Audio Reverb Zone.

**Игрок:** `ComfortLocomotion` + CharacterController на XR Origin, `VignetteRing` на камере, `HeightCalibration`, один `ComfortSettings` в сцене.

## 4. Управление

- Достать: рука к поясу справа, grip.
- Взвести: вторая рука, щепоть (trigger) на шпоре, тянуть назад; или стик главной руки назад.
- Стрелять: trigger главной руки (только при взведённом).
- Открыть: пальцем нажать защёлку, затем grip на барабане и тянуть влево; закрыть: подвести к раме.
- Крутить барабан: щепоть (trigger) на открытом барабане, круговое движение.
- Выбросить гильзы: толкнуть шток пальцем, встряхнуть или поднять ствол (живые).
- Зарядить: патрон со стола двумя пальцами (grip), донцем к устью каморы. Ствол держать горизонтально или вниз: при стволе вверх патроны выпадают.

## 5. MVP и этапы

MVP: одна дорожка, 6 камор, взвод второй рукой, патроны на столе, три мишени со счётом.

| Этап | Что | Готово, когда |
|---|---|---|
| 1 | Риг, руки, `HandRef`, хаптика, слои | куб берётся и бросается |
| 2 | Револьвер в руке, `Holster`, `TwoHandAim` | доставать из кобуры приятно, нет дрожи |
| 3 | `Hammer`, `RevolverTrigger`, `Bullet`, `Target` | выстрел только после взвода, очки идут |
| 4 | `Cylinder`, `Chamber`, `Cartridge`, `AmmoTable` | полный цикл перезарядки без кнопок |
| 5 | Отдача, дым, звук, двурукий хват | оружие «весит» |
| 6 | Локомоция, виньетка, snap-turn, высота, настройки | 5 минут игры без тошноты |
| 7 | Профилирование Quest, полировка | 90 Гц, ≤ 200 draw calls |

## 6. Что проверить при первом запуске

1. **API XRI 3.x:** `HapticImpulsePlayer.SendHapticImpulse`, `XRInputButtonReader.ReadValue`, `GetAttachTransform`, `StartManualInteraction`, `CancelInteractableSelection`. В 2.x имена другие.
2. **Сокет и мультихват:** Select Mode Multiple + сокет. Если рука не забирает револьвер из кобуры, проверь `Holster.OnRevolverSelected` и Interaction Layers.
3. **Порядок обновления:** `TwoHandAim` пишет позу в `LateUpdate` и `onBeforeRender`. Если видишь дрожь рук, оставь один из двух вызовов.
4. **Мелкие тела:** патроны R 5 мм на 90 Гц. Если пролетают сквозь стол, увеличь толщину столешницы и Solver Iterations патрона до 12–16.
5. **Большой палец:** на Touch-контроллерах он эмулируется стиком (`HammerThumb`). Настоящий палец возможен только с hand tracking.
6. **Виньетка:** шейдер `Sprites/Default` должен быть в Always Included Shaders (по умолчанию он там есть).
7. **Левша:** доминантная рука задаётся в `ComfortSettings`; кобура зеркалится, движение уходит на стик другой руки. Стики в Input Actions назначить вручную.
