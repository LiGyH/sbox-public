# 3. Система сцен (Scene System)

## Что такое сцена?

**Сцена (Scene)** — это контейнер для всех игровых объектов. Можно думать о ней как об **уровне** или **карте** в игре. В сцене находятся:

- Все игровые объекты (GameObject)
- Физический мир (PhysicsWorld)
- Визуальный мир (SceneWorld)
- Освещение, камеры, звуки

### Аналогия
Сцена — это как **сцена в театре**: у неё есть декорации (объекты), освещение, актёры (игроки) и правила (физика).

---

## Работа с активной сценой

В каждый момент времени есть **одна активная сцена**. Доступ к ней — через `Game.ActiveScene` или из любого компонента через свойство `Scene`:

```csharp
public sealed class SceneExample : Component
{
    protected override void OnAwake()
    {
        // Текущая сцена (из компонента)
        Scene currentScene = Scene;
        
        // Глобальный доступ к активной сцене
        Scene activeScene = Game.ActiveScene;
        
        // Физический мир этой сцены
        PhysicsWorld physics = Scene.PhysicsWorld;
        
        // Визуальный мир
        SceneWorld sceneWorld = Scene.SceneWorld;
    }
}
```

---

## Создание объектов в сцене

### Создание пустого объекта

```csharp
public sealed class SpawnExample : Component
{
    [Property]
    public GameObject PrefabToSpawn { get; set; }
    
    protected override void OnUpdate()
    {
        // Спавн по нажатию кнопки
        if ( Input.Pressed( "Attack1" ) )
        {
            SpawnObject();
        }
    }
    
    void SpawnObject()
    {
        // Способ 1: Создать пустой объект
        var obj = new GameObject( true, "СозданныйОбъект" );
        obj.WorldPosition = WorldPosition + WorldRotation.Forward * 100;
        
        // Добавить компоненты
        var renderer = obj.Components.Create<ModelRenderer>();
        renderer.Model = Model.Load( "models/dev/box.vmdl" );
        
        var collider = obj.Components.Create<BoxCollider>();
        collider.Scale = new Vector3( 32, 32, 32 );
        
        var body = obj.Components.Create<Rigidbody>();
        body.Gravity = true;
        
        // Придаём импульс вперёд
        body.Velocity = WorldRotation.Forward * 500;
    }
}
```

### Создание из префаба (Prefab)

Префаб — это заранее сохранённый объект со всеми компонентами:

```csharp
public sealed class PrefabSpawnExample : Component
{
    // Ссылка на префаб (перетащите в редакторе)
    [Property]
    public PrefabFile EnemyPrefab { get; set; }
    
    [Property]
    public PrefabFile BulletPrefab { get; set; }
    
    public void SpawnEnemy( Vector3 position )
    {
        if ( EnemyPrefab == null ) return;
        
        // Создаём объект из префаба
        var enemy = SceneUtility.Instantiate( EnemyPrefab, position, Rotation.Identity );
        
        Log.Info( $"Заспавнен враг на позиции: {position}" );
    }
    
    public void SpawnBullet()
    {
        if ( BulletPrefab == null ) return;
        
        var bullet = SceneUtility.Instantiate(
            BulletPrefab,
            WorldPosition + WorldRotation.Forward * 50,
            WorldRotation
        );
        
        // Настройка после спавна
        var bulletComponent = bullet.Components.Get<BulletComponent>();
        if ( bulletComponent != null )
        {
            bulletComponent.Speed = 2000f;
            bulletComponent.Damage = 25f;
        }
    }
}
```

---

## Поиск объектов в сцене

### Поиск по компонентам

```csharp
public sealed class FindInSceneExample : Component
{
    protected override void OnAwake()
    {
        // Найти ВСЕ объекты с компонентом EnemyAI
        var allEnemies = Scene.GetAllComponents<EnemyAI>();
        Log.Info( $"Найдено врагов: {allEnemies.Count()}" );
        
        // Найти ПЕРВЫЙ объект с компонентом
        var firstEnemy = Scene.GetAllComponents<EnemyAI>().FirstOrDefault();
        
        // Найти объекты определённого типа в радиусе
        float searchRadius = 500f;
        var nearbyEnemies = Scene.GetAllComponents<EnemyAI>()
            .Where( e => e.WorldPosition.Distance( WorldPosition ) < searchRadius )
            .ToList();
        
        Log.Info( $"Врагов в радиусе {searchRadius}: {nearbyEnemies.Count}" );
        
        // Найти ближайший объект
        var closest = Scene.GetAllComponents<EnemyAI>()
            .OrderBy( e => e.WorldPosition.Distance( WorldPosition ) )
            .FirstOrDefault();
        
        if ( closest != null )
        {
            Log.Info( $"Ближайший враг: {closest.GameObject.Name}" );
        }
    }
}
```

### Поиск по тегам

```csharp
public sealed class TagSearchExample : Component
{
    protected override void OnAwake()
    {
        // Получить все объекты в сцене
        var allObjects = Scene.GetAllObjects( true );
        
        // Фильтрация по тегам
        var enemies = allObjects.Where( x => x.Tags.Has( "enemy" ) );
        var players = allObjects.Where( x => x.Tags.Has( "player" ) );
        
        foreach ( var enemy in enemies )
        {
            Log.Info( $"Враг: {enemy.Name} на позиции {enemy.WorldPosition}" );
        }
    }
}
```

---

## Теги (Tags)

Каждый GameObject может иметь **теги** — метки для категоризации:

```csharp
public sealed class TagExample : Component
{
    protected override void OnAwake()
    {
        // Добавить тег
        Tags.Add( "player" );
        Tags.Add( "team_red" );
        
        // Проверить тег
        if ( Tags.Has( "player" ) )
        {
            Log.Info( "Это игрок!" );
        }
        
        // Удалить тег
        Tags.Remove( "team_red" );
        
        // Переключить тег
        Tags.Toggle( "invisible" ); // Если был — удалит, если не было — добавит
    }
}
```

---

## Работа с физикой сцены

### Трассировка лучей (Raycast)

Луч — это невидимая линия, которая проверяет, что встретилось на её пути:

```csharp
public sealed class RaycastExample : Component
{
    protected override void OnUpdate()
    {
        // === Простой луч ===
        
        // Стреляем лучом из позиции объекта вперёд на 1000 единиц
        var trace = Scene.Trace
            .Ray( WorldPosition, WorldPosition + WorldRotation.Forward * 1000 )
            .Run();
        
        if ( trace.Hit )
        {
            Log.Info( $"Попали в: {trace.GameObject?.Name}" );
            Log.Info( $"Точка попадания: {trace.HitPosition}" );
            Log.Info( $"Нормаль поверхности: {trace.Normal}" );
            Log.Info( $"Расстояние: {trace.Distance}" );
        }
        
        // === Луч с фильтрами ===
        
        var filteredTrace = Scene.Trace
            .Ray( WorldPosition, WorldPosition + WorldRotation.Forward * 1000 )
            .WithTag( "solid" )          // Только объекты с тегом "solid"
            .WithoutTags( "player" )     // Игнорировать игроков
            .IgnoreGameObject( GameObject ) // Игнорировать себя
            .Run();
        
        // === Сферический луч (Sphere Cast) ===
        
        var sphereTrace = Scene.Trace
            .Sphere( 16f, WorldPosition, WorldPosition + WorldRotation.Forward * 500 )
            .Run();
        
        // === Луч-коробка (Box Cast) ===
        
        var boxTrace = Scene.Trace
            .Box( new Vector3( 32, 32, 64 ), WorldPosition, WorldPosition + Vector3.Down * 200 )
            .Run();
    }
}
```

### Пример: Стрельба оружием

```csharp
public sealed class WeaponShoot : Component
{
    [Property] public float Damage { get; set; } = 25f;
    [Property] public float Range { get; set; } = 5000f;
    [Property] public float FireRate { get; set; } = 0.1f; // Секунд между выстрелами
    [Property] public GameObject MuzzleFlash { get; set; }  // Точка дула
    
    private TimeUntil nextFire;
    
    protected override void OnUpdate()
    {
        if ( !Input.Down( "Attack1" ) ) return;
        if ( nextFire > 0 ) return;
        
        nextFire = FireRate;
        Fire();
    }
    
    void Fire()
    {
        // Определяем начало и конец луча
        Vector3 startPos = MuzzleFlash?.WorldPosition ?? WorldPosition;
        Vector3 endPos = startPos + WorldRotation.Forward * Range;
        
        // Стреляем лучом
        var trace = Scene.Trace
            .Ray( startPos, endPos )
            .IgnoreGameObject( GameObject )   // Не попадать в себя
            .WithoutTags( "trigger" )         // Игнорировать триггеры
            .Run();
        
        if ( trace.Hit )
        {
            // Наносим урон если у цели есть компонент здоровья
            var health = trace.GameObject?.Components.Get<HealthComponent>();
            if ( health != null )
            {
                health.TakeDamage( Damage );
            }
            
            Log.Info( $"Попадание в {trace.GameObject?.Name} на расстоянии {trace.Distance:F1}" );
        }
    }
}
```

---

## Загрузка сцен

```csharp
public sealed class SceneLoadExample : Component
{
    [Property]
    public SceneFile NextScene { get; set; }
    
    public void LoadNextLevel()
    {
        if ( NextScene == null )
        {
            Log.Warning( "Не указана следующая сцена!" );
            return;
        }
        
        // Загружаем новую сцену
        Game.ActiveScene.Load( NextScene );
    }
}
```

---

## Практический пример: Менеджер волн врагов

```csharp
/// <summary>
/// Спавнит волны врагов с нарастающей сложностью.
/// </summary>
[Title( "Менеджер волн" )]
[Category( "Игровая логика" )]
[Icon( "waves" )]
public sealed class WaveManager : Component
{
    [Property] public PrefabFile EnemyPrefab { get; set; }
    [Property] public List<GameObject> SpawnPoints { get; set; } = new();
    [Property] public float TimeBetweenWaves { get; set; } = 10f;
    [Property] public int EnemiesPerWave { get; set; } = 5;
    [Property] public float DifficultyMultiplier { get; set; } = 1.2f;
    
    public int CurrentWave { get; private set; }
    public int EnemiesAlive { get; private set; }
    
    private TimeUntil nextWave;
    private bool waveActive;
    
    protected override void OnAwake()
    {
        CurrentWave = 0;
        nextWave = 5f; // Первая волна через 5 секунд
    }
    
    protected override void OnUpdate()
    {
        // Считаем живых врагов
        EnemiesAlive = Scene.GetAllComponents<EnemyAI>().Count();
        
        // Если волна не активна и таймер истёк — запускаем новую
        if ( !waveActive && nextWave <= 0 )
        {
            StartNextWave();
        }
        
        // Если все враги убиты — волна завершена
        if ( waveActive && EnemiesAlive == 0 )
        {
            WaveCompleted();
        }
    }
    
    void StartNextWave()
    {
        CurrentWave++;
        waveActive = true;
        
        int enemiesToSpawn = (int)(EnemiesPerWave * MathF.Pow( DifficultyMultiplier, CurrentWave - 1 ));
        
        Log.Info( $"=== Волна {CurrentWave}! Врагов: {enemiesToSpawn} ===" );
        
        for ( int i = 0; i < enemiesToSpawn; i++ )
        {
            SpawnEnemy();
        }
    }
    
    void SpawnEnemy()
    {
        if ( EnemyPrefab == null || SpawnPoints.Count == 0 ) return;
        
        // Выбираем случайную точку спавна
        var spawnPoint = SpawnPoints[Random.Shared.Next( SpawnPoints.Count )];
        
        // Спавним врага
        var enemy = SceneUtility.Instantiate( EnemyPrefab, spawnPoint.WorldPosition, spawnPoint.WorldRotation );
    }
    
    void WaveCompleted()
    {
        waveActive = false;
        nextWave = TimeBetweenWaves;
        
        Log.Info( $"Волна {CurrentWave} завершена! Следующая через {TimeBetweenWaves} секунд." );
    }
}
```

---

## Следующий шаг

Перейдите к разделу **[UI система — Razor](../04-UI-Razor/README.md)** для изучения создания интерфейсов.
