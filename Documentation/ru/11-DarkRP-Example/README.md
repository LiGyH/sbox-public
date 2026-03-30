# 11. Пример проекта: DarkRP

## Что мы создаём?

**DarkRP** — это популярный игровой режим, где игроки выбирают **профессии** (полицейский, вор, торговец и т.д.), зарабатывают деньги, покупают предметы и взаимодействуют друг с другом.

В этом разделе мы пошагово создадим **упрощённую версию DarkRP** в s&box, покрывая:

- 🎭 Систему профессий (Jobs)
- 💰 Экономику (деньги, зарплата)
- 🏪 Магазин и покупки
- 🚪 Двери (покупка, блокировка)
- 🔫 Оружие
- 💬 Чат и команды
- 📊 HUD (интерфейс)
- 🌐 Полная сетевая синхронизация

---

## Структура проекта

```
DarkRP/
├── code/
│   ├── Assembly.cs
│   ├── Systems/                    # Основные системы
│   │   ├── GameManager.cs          # Главный менеджер игры
│   │   ├── JobSystem.cs            # Система профессий
│   │   ├── MoneySystem.cs          # Экономика
│   │   └── DoorSystem.cs           # Система дверей
│   ├── Player/                     # Код игрока
│   │   ├── DarkRPPlayer.cs         # Компонент игрока
│   │   └── PlayerMovement.cs       # Движение
│   ├── Entities/                   # Игровые сущности
│   │   ├── ShopNPC.cs              # NPC-продавец
│   │   ├── MoneyPrinter.cs         # Принтер денег
│   │   └── DroppedMoney.cs         # Выброшенные деньги
│   ├── Resources/                  # Определения ресурсов
│   │   └── JobDefinition.cs        # GameResource для профессии
│   ├── UI/                         # Интерфейс
│   │   ├── Hud/
│   │   │   ├── DarkRPHud.razor
│   │   │   └── DarkRPHud.razor.scss
│   │   ├── JobMenu/
│   │   │   ├── JobMenu.razor
│   │   │   └── JobMenu.razor.scss
│   │   ├── Shop/
│   │   │   ├── ShopMenu.razor
│   │   │   └── ShopMenu.razor.scss
│   │   └── Chat/
│   │       ├── ChatBox.razor
│   │       └── ChatBox.razor.scss
│   └── Weapons/                    # Оружие
│       └── BaseWeapon.cs
├── models/
├── materials/
├── sounds/
└── scenes/
    └── darkrp_downtown.scene
```

## Содержание раздела

1. **[Определение профессий (GameResource)](01-JobDefinition.md)**
2. **[Компонент игрока](02-PlayerComponent.md)**
3. **[Система профессий и экономики](03-JobAndMoney.md)**
4. **[Игровые сущности](04-Entities.md)**
5. **[Интерфейс (HUD, меню)](05-UI.md)**
6. **[Система дверей](06-Doors.md)**
7. **[Оружие](07-Weapons.md)**
8. **[Главный менеджер игры](08-GameManager.md)**

---

Начните с **[Определение профессий →](01-JobDefinition.md)**
