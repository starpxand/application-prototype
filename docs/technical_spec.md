# Техническое описание прототипа приложения

> **Задание 1** лабораторной работы №5
> Документ определяет основной функционал и логику работы создаваемого прототипа

---

## Общее описание

**Название:** GTA VI Web Prototype — Vice (Police Edition)  
**Тип:** Браузерная 3D-игра в жанре «открытый мир» (single-page application)  
**Технологический стек:** HTML5, CSS3, JavaScript ES6 Modules, Three.js 0.158.0  
**Зависимости:** только Three.js (CDN, без установки)  
**Запуск:** открытие HTML-файла в браузере, без серверной части

---

## 1. Архитектура приложения

Приложение реализовано как единый HTML-файл с ES6-модулями. Структурно делится на следующие слои:

```
┌───────────────────────────────────────────┐
│            HTML/CSS Layer                 │
│  (HUD-оверлей, стартовый экран, стили)    │
├───────────────────────────────────────────┤
│          Game Loop (requestAnimationFrame)│
│  animate() → update systems → render()    │
├──────────────┬────────────────────────────┤
│  Render      │  Game Systems              │
│  Three.js    │  • City Generator          │
│  WebGL       │  • Player Controller       │
│  Scene       │  • NPC AI                  │
│  Camera      │  • Vehicle Physics         │
│  Renderer    │  • Weapon & Raycasting     │
│              │  • Particle System         │
└──────────────┴────────────────────────────┘
```

---

## 2. Игровые системы

### 2.1 Генератор города (Procedural City)

**Назначение:** создание игрового мира при каждом запуске.

**Параметры:**
- `blockSize = 50` — размер квартала
- `roadWidth = 14` — ширина дороги
- `cityRadius = 200` — радиус города

**Логика:**
1. Создаётся плоскость земли 1000×1000 с процедурной текстурой травы
2. По сетке формируются горизонтальные и вертикальные дороги с жёлтыми полосами разметки
3. В каждом блоке с вероятностью 70% генерируется здание одного из трёх типов:
   - `BoxGeometry` — прямоугольное здание
   - `CylinderGeometry` — цилиндрическое здание
   - Многоуровневое (BoxGeometry + крыша)
4. В пустых блоках (30%) спавнится мини-парк из 2–5 деревьев
5. Дерево: ствол `CylinderGeometry(0.3, 0.5, 4)` + крона `IcosahedronGeometry(2.5)`

**Текстуры (Canvas API):**
- `road` — асфальт тёмного цвета с двойной жёлтой разметкой по центру
- `building` — фасад с сеткой окон (тёмные + светящиеся жёлтые/розовые)
- `grass` — травяное покрытие

---

### 2.2 Контроллер игрока (Player Controller)

**Вид:** от первого лица (FPS)  
**Система ввода:** PointerLockControls (Three.js addon) + KeyboardEvent

**Состояния и скорости:**
```
walk:  speed = 10 units/sec
run:   speed = 20 units/sec  (SHIFT удержан)
jump:  velocityY = +10,  gravity = -20
```

**Коллизии:** проверка по массиву `objects[]` через `Box3.intersectsBox()`

---

### 2.3 ИИ пешеходов (NPC AI)

**Количество:** 180 NPC

**Состояние `walk`:**
- Следует по массиву `waypoints[]` (5 случайных точек в радиусе 40 ед.)
- Скорость: `speed = 2.0–3.5 units/sec`
- При достижении waypoint: `wpIdx = (wpIdx + 1) % waypoints.length`
- Анимация ног: синусоидальное покачивание `legL.rotation.x = sin(t×5) × 0.3`

**Состояние `panic`:**
- Активируется при дистанции до игрока < 12 единиц
- Убегает от позиции игрока по вектору `(npc.pos - player.pos).normalize()`
- Скорость: `speed × 3.5`
- `panicTimer = 3.0 sec` → после истечения возврат в `walk`

**Состояние `dead`:**
- Активируется при попадании пули или наезде автомобиля
- NPC медленно «падает» (`rotation.x → π/2`) и опускается под землю
- Удаляется из массива коллизий `objects[]`

---

### 2.4 Транспортная система (Vehicle Physics)

**Количество:** 15 автомобилей (стилизованы под полицейские с красно-синими мигалками)

**Структура объекта Car:**
```javascript
{
  mesh: THREE.Group,    // кузов + крыша + колёса + фары + PointLight (red/blue)
  speed: 0,            // текущая скорость
  maxSpeed: 40,        // максимальная скорость
  angle: 0,            // направление (rad)
  driftAngle: 0,       // угол дрифта
  driftVelocity: Vector3,
  isOccupied: false,   // занята ли игроком
  sirenTimer: 0        // таймер мигалки
}
```

**Физика:**
- Ускорение: `speed += 20 * delta` (W)
- Торможение: `speed -= 25 * delta` (S)
- Трение качения: `speed *= 0.97` (каждый кадр)
- Поворот: `angle += turnSpeed * delta * sign(speed)` (A/D)
- Дрифт (ПРОБЕЛ при speed > 10): `turnSpeed *= 1.8`, дым из-под задних колёс

**Взаимодействие:**
- Клавиша `F` при `distanceTo(car) < 4` → посадка, `gunGroup.visible = false`
- Клавиша `F` повторно → высадка рядом с дверью (`car.mesh.position + offset`)
- Сбивание NPC: `carBox.expandByScalar(0.4)` + `killNPC()` + `speed *= 0.85`

**Мигалки:** `sirenTimer += delta × 15`; чётный кадр — красный свет, нечётный — синий (PointLight intensity 0/15)

---

### 2.5 Система оружия и стрельбы

**Модель оружия** (неоновая цветовая схема, `camera.add(gunGroup)`):
- Корпус: `BoxGeometry(0.06, 0.12, 0.35)`, цвет `#ff007f` (розовый)
- Ствол: `CylinderGeometry(0.015, 0.015, 0.3)`, цвет `#00ffff` (бирюзовый)
- Приклад/рукоять: цвет `#aa00ff` (фиолетовый)
- Магазин: цвет `#ffea00` (жёлтый)
- Прицел: белый + красное стекло (прозрачное)
- Вспышка: `PlaneGeometry(0.3)`, видима 55 мс при выстреле

**Стрельба (Raycasting):**
```
1. THREE.Raycaster из центра камеры (Vector2(0,0))
2. Приоритет: npcMeshes[] → objects[]
3. Попадание NPC:  killNPC(group) + spawnNPCBlood(point)
4. Попадание объект: holeMesh(12с) + 5 искр(0.18с)
```

**Параметры:**
- `magazineSize = 30`, `AUTO_FIRE_RATE = 0.09с` (~11 выстр./с)
- Перезарядка: автоматическая через 1.2 с при пустом магазине
- Отдача: `gunGroup.rotation.x -= 0.05`, возврат через `setTimeout(50ms)`

---

### 2.6 Система частиц (Particle System)

| Эффект | Геометрия | Продолжительность |
|--------|-----------|------------------|
| Искры (Spark) | `PlaneGeometry(0.1–0.22)` ×5, оранжевые | 0.18 сек, статичны |
| Кровь (Blood) | `PlaneGeometry(0.25)` ×6, красные | 0.7 сек, падение вниз |
| Гильзы (Shell) | `CylinderGeometry(0.015, 0.015, 0.06)`, латунь | 1.2 сек, баллистика + гравитация |
| Дым (Smoke) | `PlaneGeometry(1, 1)`, серый, прозрачный | ~2 сек, подъём + масштаб |
| Отверстие (Hole) | `PlaneGeometry(0.12, 0.12)`, чёрное | 12 сек (постоянное) |

---

### 2.7 HUD-интерфейс

HUD реализован как HTML-слой поверх WebGL-canvas (`position: absolute`):

![HUD-интерфейс](screenshots/HUD-интерфейс.png)

- **Деньги** (`#money`): зелёный текст 36px, правый верхний угол, `$5,000,000`
- **Розыск** (`#wanted`): красный текст, пять звёзд ⭐⭐⭐⭐⭐, под деньгами
- **Прицел** (`#crosshair`): белая точка 6×6 px в центре экрана
- **Патроны** (`#ammo`): белый жирный текст 32px, правый нижний угол
- **Подсказка** (`#interactionHint`): бирюзовый блок «Нажми F сесть» при dist < 4 ед.
- **Счётчик убийств**: красный Impact-текст «KILLED! [N]» в центре, исчезает через 800 мс

---

## 3. Игровой цикл (Game Loop)

```javascript
function animate() {
  requestAnimationFrame(animate);
  const delta = Math.min(clock.getDelta(), 0.1); // защита от freeze-spike

  if (isShooting && !isDriving) {
    autoFireTimer -= delta;
    if (autoFireTimer <= 0) { fireBullet(); autoFireTimer = AUTO_FIRE_RATE; }
  }

  updateNPCs(delta);        // ИИ пешеходов (FSM + анимация ног)
  updateBullets(delta);     // физика снарядов + вспышка
  updateParticles(delta);   // частицы (кровь, искры, дым)

  if (controls.isLocked) {
    updateSirenLights(delta); // мигалки полицейских авто
    if (isDriving) updateCar(delta); // физика вождения
    else updatePlayer(delta);        // движение пешком + коллизии
    checkInteraction();              // подсказка F у машины
  }

  renderer.render(scene, camera);
}
```

---

## 4. Визуальный стиль

**Цветовая схема Miami Vice:**
- Фон неба / туман: `#ff8c69` (оранжево-розовый закат), `FogExp2(density=0.0035)`
- Освещение: `AmbientLight(#ffccff, 0.5)` + `DirectionalLight(#ffaa55, 1.8)`
- Тональное отображение: `ACESFilmicToneMapping`, exposure = 1.2
- Тени: `PCFSoftShadowMap`, shadowMap 2048×2048, только DirectionalLight
- Звёзды: 1000 `THREE.Points` на полусфере радиуса 350 единиц

---

## 5. Требования к окружению

- Браузер с поддержкой **WebGL 1.0+** (Chrome 90+, Firefox 88+, Edge 90+)
- Рекомендуется дискретная видеокарта для стабильных 60 FPS
- JavaScript должен быть включён
- Интернет-соединение для загрузки Three.js с CDN (при первом запуске)
