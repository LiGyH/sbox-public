# DarkRP: Шаг 8 — Главный менеджер игры

## GameManager.cs — Центральный контроллер

Этот компонент управляет **всей игрой**: спавн игроков, подключения, основные правила.

```csharp
// code/Systems/GameManager.cs

using Sandbox;
using System.Linq;

/// <summary>
/// Главный менеджер DarkRP.
/// Управляет подключениями, спавном, основными правилами.
/// Добавьте на объект в сцене — он должен быть ОДИН.
/// </summary>
[Title( "DarkRP Менеджер" )]
[Category( "DarkRP" )]
[Icon( "settings" )]
public sealed class DarkRPGameManager : Component, Component.INetworkListener
{
    // === НАСТРОЙКИ ===
    
    [Property] public PrefabFile PlayerPrefab { get; set; }
    [Property] public JobDefinition DefaultJob { get; set; }
    [Property] public string ServerName { get; set; } = "DarkRP Сервер";
    [Property] public int StartMoney { get; set; } = 500;
    
    // === ССЫЛКИ НА СИСТЕМЫ ===
    
    public JobSystem Jobs { get; private set; }
    public MoneySystem Economy { get; private set; }
    
    // ============================
    // ИНИЦИАЛИЗАЦИЯ
    // ============================
    
    protected override void OnAwake()
    {
        // Находим подсистемы
        Jobs = Scene.GetAllComponents<JobSystem>().FirstOrDefault();
        Economy = Scene.GetAllComponents<MoneySystem>().FirstOrDefault();
        
        Log.Info( $"=== {ServerName} запущен ===" );
    }
    
    // ============================
    // СЕТЕВЫЕ СОБЫТИЯ
    // ============================
    
    /// <summary>Новый игрок подключился</summary>
    void INetworkListener.OnConnected( Connection connection )
    {
        Log.Info( $"Подключился: {connection.DisplayName}" );
        
        // Спавним объект игрока
        SpawnPlayer( connection );
    }
    
    /// <summary>Игрок отключился</summary>
    void INetworkListener.OnDisconnected( Connection connection )
    {
        Log.Info( $"Отключился: {connection.DisplayName}" );
        
        // Объект автоматически удаляется при отключении владельца
    }
    
    // ============================
    // СПАВН ИГРОКА
    // ============================
    
    void SpawnPlayer( Connection connection )
    {
        if ( PlayerPrefab == null )
        {
            Log.Error( "PlayerPrefab не назначен!" );
            return;
        }
        
        // Находим точку спавна
        Vector3 spawnPos = GetSpawnPosition();
        Rotation spawnRot = GetSpawnRotation();
        
        // Создаём объект из префаба
        var playerObj = SceneUtility.Instantiate( PlayerPrefab, spawnPos, spawnRot );
        
        // Регистрируем в сети с владельцем
        playerObj.NetworkSpawn( connection );
        
        // Настраиваем компонент игрока
        var player = playerObj.Components.Get<DarkRPPlayer>();
        if ( player != null )
        {
            player.PlayerName = connection.DisplayName;
            player.Money = StartMoney;
            
            // Назначаем стандартную профессию
            if ( DefaultJob != null )
            {
                player.SetJob( DefaultJob );
            }
        }
        
        Log.Info( $"Игрок {connection.DisplayName} заспавнен на {spawnPos}" );
    }
    
    Vector3 GetSpawnPosition()
    {
        var spawnPoints = Scene.GetAllComponents<SpawnPoint>().ToList();
        
        if ( spawnPoints.Count == 0 )
        {
            Log.Warning( "Нет точек спавна! Используем (0, 0, 100)" );
            return new Vector3( 0, 0, 100 );
        }
        
        // Случайная точка
        var random = spawnPoints[Random.Shared.Next( spawnPoints.Count )];
        return random.WorldPosition;
    }
    
    Rotation GetSpawnRotation()
    {
        var spawnPoints = Scene.GetAllComponents<SpawnPoint>().ToList();
        
        if ( spawnPoints.Count == 0 )
            return Rotation.Identity;
        
        var random = spawnPoints[Random.Shared.Next( spawnPoints.Count )];
        return random.WorldRotation;
    }
    
    // ============================
    // РЕСПАВН
    // ============================
    
    /// <summary>Респавнить игрока после смерти</summary>
    public void RespawnPlayer( DarkRPPlayer player )
    {
        if ( player == null ) return;
        
        Vector3 spawnPos = GetSpawnPosition();
        player.Respawn( spawnPos );
        
        Log.Info( $"{player.PlayerName} воскрешён" );
    }
}
```

---

## Настройка сцены

### Какие объекты нужны в сцене

```
Сцена "darkrp_downtown"
│
├── 🎮 GameManager (GameObject)
│   ├── DarkRPGameManager (компонент)
│   ├── JobSystem (компонент)
│   └── MoneySystem (компонент)
│
├── 📷 MainCamera (GameObject)
│   └── CameraComponent
│
├── ☀️ Sun (GameObject)
│   └── DirectionalLight
│
├── 🗺 Map (GameObject)
│   └── MapInstance (загружает карту)
│
├── 🚩 SpawnPoint_1 (GameObject)
│   └── SpawnPoint
│
├── 🚩 SpawnPoint_2 (GameObject)
│   └── SpawnPoint
│
├── 🏪 WeaponShop (GameObject)
│   ├── ModelRenderer (модель NPC)
│   └── ShopNPC (компонент)
│
├── 🚪 Door_1 (GameObject)
│   ├── ModelRenderer
│   ├── ModelCollider
│   └── DarkRPDoor
│
└── 🖥 HUD (GameObject)
    ├── ScreenPanel
    ├── DarkRPHud
    └── JobMenu
```

### Шаги создания сцены

1. **Создайте новую сцену** в редакторе
2. **Добавьте объект GameManager** и прикрепите:
   - `DarkRPGameManager`
   - `JobSystem` (заполните список профессий)
   - `MoneySystem`
3. **Создайте префаб игрока:**
   - GameObject "Player"
     - `DarkRPPlayer`
     - `CharacterController`
     - `SkinnedModelRenderer` (модель citizen)
     - Дочерний объект "Camera" с `CameraComponent`
   - Сохраните как `player.prefab`
4. **Добавьте точки спавна** (SpawnPoint) по карте
5. **Добавьте HUD объект** с `ScreenPanel`, `DarkRPHud`, `JobMenu`
6. **Расставьте двери** с компонентом `DarkRPDoor`
7. **Расставьте NPC** с компонентом `ShopNPC`

---

## Итог: что мы создали

| Файл | Описание |
|------|----------|
| `JobDefinition.cs` | GameResource для определения профессий |
| `DarkRPPlayer.cs` | Компонент игрока (здоровье, деньги, профессия) |
| `JobSystem.cs` | Система управления профессиями |
| `MoneySystem.cs` | Экономика (переводы, штрафы) |
| `DroppedMoney.cs` | Деньги на земле |
| `MoneyPrinter.cs` | Принтер денег |
| `ShopNPC.cs` | NPC-продавец |
| `DarkRPDoor.cs` | Покупаемые двери |
| `BaseWeapon.cs` | Базовое оружие |
| `DarkRPGameManager.cs` | Главный менеджер игры |
| `DarkRPHud.razor` + `.razor.scss` | Игровой HUD |
| `JobMenu.razor` + `.razor.scss` | Меню выбора профессии |

### Что можно добавить дальше

- 🏠 **Система домов** — покупка зданий целиком
- 📻 **Голосование** — на мэра, на кик
- 🔒 **Взлом замков** — для вора (мини-игра)
- 🏪 **Свои магазины** — игроки могут продавать предметы
- 📦 **Инвентарь** — сетка предметов
- 🚗 **Транспорт** — машины, вертолёты
- 📜 **Система законов** — мэр может создавать законы
- 🎵 **Система радио** — музыка в зданиях
- 💊 **Наркотики** — вор может готовить и продавать

---

## Поздравляем!

Вы создали базовую версию DarkRP в s&box! 🎉

Этот пример покрывает все основные системы:
- ✅ Компоненты и GameObject
- ✅ Сетевая синхронизация ([Sync], [Broadcast])
- ✅ UI с Razor и SCSS (.razor.scss!)
- ✅ Физика и трассировка лучей
- ✅ Система ввода (Input)
- ✅ GameResource (пользовательские ассеты)
- ✅ Интерфейсы взаимодействия

Возвращайтесь к **[Главной странице документации](../README.md)** для повторения любого раздела.
