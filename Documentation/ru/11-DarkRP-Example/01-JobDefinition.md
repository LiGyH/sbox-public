# DarkRP: Шаг 1 — Определение профессий (JobDefinition)

## Что такое GameResource?

`GameResource` — это файл-ассет, который можно создать и настроить прямо в **редакторе s&box** без программирования. Для нашего DarkRP мы создадим **JobDefinition** — определение профессии.

---

## Код: JobDefinition.cs

```csharp
// code/Resources/JobDefinition.cs

using Sandbox;
using System.Collections.Generic;

/// <summary>
/// Определение профессии для DarkRP.
/// Создаётся как файл .job в редакторе.
/// 
/// Примеры:
///   citizen.job    — Гражданин
///   police.job     — Полицейский
///   thief.job      — Вор
///   medic.job      — Медик
///   dealer.job     — Торговец оружием
///   mayor.job      — Мэр
/// </summary>
[GameResource( "Job Definition", "job", "Определение профессии DarkRP" )]
public class JobDefinition : GameResource
{
    // === ОСНОВНАЯ ИНФОРМАЦИЯ ===
    
    /// <summary>Название профессии (отображается в меню и над головой)</summary>
    [Property, Title( "Название" )]
    public string JobName { get; set; } = "Гражданин";
    
    /// <summary>Описание профессии (показывается в меню выбора)</summary>
    [Property, TextArea, Title( "Описание" )]
    public string Description { get; set; } = "Обычный гражданин города.";
    
    /// <summary>Цвет профессии (для имени над головой, в чате и т.д.)</summary>
    [Property, Title( "Цвет" )]
    public Color JobColor { get; set; } = Color.White;
    
    /// <summary>Категория (Гражданские, Силовые, Криминальные)</summary>
    [Property, Title( "Категория" )]
    public string Category { get; set; } = "Гражданские";
    
    // === ЭКОНОМИКА ===
    
    /// <summary>Зарплата (деньги каждые SalaryInterval секунд)</summary>
    [Property, Title( "Зарплата" ), Range( 0, 1000 )]
    public int Salary { get; set; } = 50;
    
    /// <summary>Интервал выплаты зарплаты (в секундах)</summary>
    [Property, Title( "Интервал зарплаты" ), Range( 10, 300 )]
    public float SalaryInterval { get; set; } = 60f;
    
    /// <summary>Начальные деньги при выборе профессии</summary>
    [Property, Title( "Начальные деньги" )]
    public int StartingMoney { get; set; } = 500;
    
    // === ОГРАНИЧЕНИЯ ===
    
    /// <summary>Максимальное количество игроков с этой профессией</summary>
    [Property, Title( "Макс. игроков" ), Range( 0, 32 )]
    public int MaxPlayers { get; set; } = 0; // 0 = без ограничений
    
    /// <summary>Может ли использовать оружие</summary>
    [Property, Title( "Может носить оружие" )]
    public bool CanUseWeapons { get; set; } = false;
    
    /// <summary>Может ли покупать двери</summary>
    [Property, Title( "Может покупать двери" )]
    public bool CanOwnDoors { get; set; } = true;
    
    /// <summary>Может ли арестовывать (для полиции)</summary>
    [Property, Title( "Может арестовывать" )]
    public bool CanArrest { get; set; } = false;
    
    // === ВНЕШНИЙ ВИД ===
    
    /// <summary>Модель персонажа</summary>
    [Property, Title( "Модель" )]
    public Model PlayerModel { get; set; }
    
    /// <summary>Стартовое оружие (список ResourcePath)</summary>
    [Property, Title( "Стартовое снаряжение" )]
    public List<string> StartingEquipment { get; set; } = new();
}
```

## Как создать файл профессии

1. В редакторе s&box: ПКМ в Assets → Create → Job Definition
2. Назовите файл (например, `citizen.job`)
3. Заполните поля в инспекторе

### Примеры профессий

#### citizen.job (Гражданин)
```
JobName: "Гражданин"
Description: "Обычный гражданин города. Может заниматься чем угодно."
JobColor: (1, 1, 1)  — белый
Category: "Гражданские"
Salary: 50
CanUseWeapons: false
CanOwnDoors: true
```

#### police.job (Полицейский)
```
JobName: "Полицейский"
Description: "Защищает закон и порядок. Может арестовывать нарушителей."
JobColor: (0.2, 0.4, 1.0)  — синий
Category: "Силовые"
Salary: 100
CanUseWeapons: true
CanArrest: true
MaxPlayers: 4
StartingEquipment: ["weapons/pistol", "weapons/stungun"]
```

#### thief.job (Вор)
```
JobName: "Вор"
Description: "Крадёт вещи и взламывает замки. Осторожно с полицией!"
JobColor: (0.5, 0, 0.5)  — фиолетовый
Category: "Криминальные"
Salary: 30
CanUseWeapons: true
CanOwnDoors: true
MaxPlayers: 4
StartingEquipment: ["tools/lockpick"]
```

#### mayor.job (Мэр)
```
JobName: "Мэр"
Description: "Глава города. Устанавливает законы и управляет бюджетом."
JobColor: (1, 0.84, 0)  — золотой
Category: "Силовые"
Salary: 200
CanUseWeapons: false
CanOwnDoors: true
MaxPlayers: 1
```

---

**Следующий шаг:** [Компонент игрока →](02-PlayerComponent.md)
