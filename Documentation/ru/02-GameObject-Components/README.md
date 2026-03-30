# 2. GameObject и система компонентов

## Что такое GameObject?

**GameObject** — это базовая единица в s&box. Каждый объект в игре (игрок, пуля, дверь, свет) — это GameObject. Сам по себе GameObject — это просто «контейнер» с позицией в мире. Всю логику ему дают **компоненты**.

### Аналогия
Представьте GameObject как **пустую коробку**:
- Без компонентов — это просто точка в пространстве
- Добавим `ModelRenderer` — коробка получает внешний вид
- Добавим `Rigidbody` — коробка подчиняется физике
- Добавим `BoxCollider` — коробка может сталкиваться с другими объектами
- Добавим свой компонент — коробка делает всё, что вы захотите

---

## Создание GameObject из кода

### Способ 1: Простое создание

```csharp
// Создаём пустой объект
var obj = new GameObject( true, "МойОбъект" );

// true — включён ли объект при создании
// "МойОбъект" — имя объекта (видно в редакторе)
```

### Способ 2: Создание с позицией

```csharp
var obj = new GameObject( true, "Куб" );
obj.WorldPosition = new Vector3( 0, 0, 100 );  // Позиция в мире
obj.WorldRotation = Rotation.FromYaw( 45 );     // Повёрнут на 45°
obj.WorldScale = new Vector3( 2, 2, 2 );        // В 2 раза больше
```

### Способ 3: Создание с компонентами

```csharp
// Создаём объект с моделью и физикой
var box = new GameObject( true, "ФизическийКуб" );
box.WorldPosition = new Vector3( 0, 0, 200 );

// Добавляем рендерер (внешний вид)
var renderer = box.Components.Create<ModelRenderer>();
renderer.Model = Model.Load( "models/dev/box.vmdl" );

// Добавляем коллайдер (форма столкновений)
var collider = box.Components.Create<BoxCollider>();
collider.Scale = new Vector3( 32, 32, 32 );

// Добавляем физическое тело
var body = box.Components.Create<Rigidbody>();
body.Gravity = true; // Куб будет падать
```

### Способ 4: Создание дочернего объекта

```csharp
// Создаём родительский объект
var parent = new GameObject( true, "Родитель" );
parent.WorldPosition = new Vector3( 0, 0, 0 );

// Создаём дочерний объект (привязан к родителю)
var child = new GameObject( true, "Ребёнок" );
child.SetParent( parent );
child.LocalPosition = new Vector3( 0, 0, 50 ); // 50 единиц выше родителя

// Теперь при перемещении parent, child будет двигаться вместе с ним!
```

---

## Иерархия объектов

GameObject образуют **дерево** (иерархию):

```
Сцена (Scene)
├── Игрок (GameObject)
│   ├── Камера (GameObject)
│   ├── Модель тела (GameObject)
│   └── Оружие (GameObject)
│       ├── Модель оружия (GameObject)
│       └── Точка вылета пуль (GameObject)
├── Враг (GameObject)
│   └── Модель (GameObject)
└── Освещение (GameObject)
```

### Навигация по иерархии

```csharp
public sealed class HierarchyExample : Component
{
    protected override void OnUpdate()
    {
        // Получить родителя
        GameObject parent = GameObject.Parent;
        
        // Получить корневой объект (самый верхний в иерархии)
        GameObject root = GameObject.Root;
        
        // Получить все дочерние объекты
        foreach ( var child in GameObject.Children )
        {
            Log.Info( $"Дочерний объект: {child.Name}" );
        }
        
        // Проверить, является ли объект корневым
        bool isRoot = GameObject.IsRoot;
        
        // Получить сцену
        Scene scene = Scene;
    }
}
```

### Поиск объектов

```csharp
public sealed class FindObjectsExample : Component
{
    protected override void OnAwake()
    {
        // Найти объект по имени во всей сцене
        var player = Scene.GetAllObjects( true )
            .FirstOrDefault( x => x.Name == "Player" );
        
        // Найти все объекты с определённым компонентом
        var allEnemies = Scene.GetAllComponents<EnemyComponent>();
        
        // Найти ближайший объект
        var nearest = Scene.GetAllComponents<PickupComponent>()
            .OrderBy( x => x.WorldPosition.Distance( WorldPosition ) )
            .FirstOrDefault();
        
        if ( nearest != null )
        {
            Log.Info( $"Ближайший пикап: {nearest.GameObject.Name}" );
            Log.Info( $"Расстояние: {nearest.WorldPosition.Distance( WorldPosition )}" );
        }
    }
}
```

---

## Система компонентов (подробно)

### Базовый компонент

Каждый компонент наследуется от `Component`:

```csharp
public sealed class MyComponent : Component
{
    // Свойства (видны в редакторе)
    [Property] public float Speed { get; set; } = 100f;
    [Property] public string Name { get; set; } = "Default";
    [Property] public Color Color { get; set; } = Color.White;
    [Property] public GameObject Target { get; set; } // Ссылка на другой объект!
    [Property] public Model MyModel { get; set; }     // Ссылка на модель
    
    // Приватные поля (НЕ видны в редакторе)
    private float timer;
    private bool isReady;
}
```

### Все атрибуты для свойств

```csharp
public sealed class AttributesExample : Component
{
    // Базовое свойство
    [Property]
    public float BasicValue { get; set; } = 10f;
    
    // С диапазоном (ползунок)
    [Property, Range( 0, 100 )]
    public float Health { get; set; } = 100f;
    
    // С шагом (ползунок с фиксированными значениями)
    [Property, Range( 0, 10 ), Step( 0.5f )]
    public float StepValue { get; set; } = 5f;
    
    // Многострочный текст
    [Property, TextArea]
    public string Description { get; set; } = "Описание объекта";
    
    // Своё имя в редакторе
    [Property, Title( "Скорость передвижения" )]
    public float MoveSpeed { get; set; } = 200f;
    
    // Подсказка при наведении
    [Property, Tooltip( "Максимальная высота прыжка в юнитах" )]
    public float JumpHeight { get; set; } = 300f;
    
    // Группировка свойств
    [Property, Group( "Движение" )]
    public float WalkSpeed { get; set; } = 200f;
    
    [Property, Group( "Движение" )]
    public float RunSpeed { get; set; } = 400f;
    
    [Property, Group( "Боевая система" )]
    public float Damage { get; set; } = 25f;
    
    [Property, Group( "Боевая система" )]
    public float AttackRange { get; set; } = 100f;
    
    // Иконка для компонента в редакторе
    // (атрибут класса, не свойства)
}

// Атрибуты класса — меняют отображение компонента в редакторе
[Title( "Контроллер врага" )]         // Название в списке компонентов
[Category( "Игровая логика" )]        // Категория
[Icon( "smart_toy" )]                  // Иконка (Material Icons)
[Description( "Управляет поведением AI врага" )]
public sealed class EnemyController : Component
{
    // ...
}
```

### Доступ к компонентам

```csharp
public sealed class ComponentAccessExample : Component
{
    protected override void OnAwake()
    {
        // === На ЭТОМ объекте ===
        
        // Получить один компонент
        var renderer = Components.Get<ModelRenderer>();
        
        // Получить один компонент (вернёт null если не найден)
        var body = Components.Get<Rigidbody>();
        if ( body != null )
        {
            body.Gravity = true;
        }
        
        // Проверить наличие компонента
        bool hasCollider = Components.Get<Collider>() != null;
        
        // Получить ВСЕ компоненты определённого типа
        var allColliders = Components.GetAll<Collider>();
        foreach ( var col in allColliders )
        {
            Log.Info( $"Найден коллайдер: {col.GetType().Name}" );
        }
        
        // === На ДОЧЕРНИХ объектах ===
        
        // Найти компонент в дочерних объектах
        var childRenderer = Components.GetInChildren<ModelRenderer>();
        
        // Найти ВСЕ компоненты в дочерних объектах
        var allChildRenderers = Components.GetAll<ModelRenderer>( FindMode.InChildren );
        
        // === На РОДИТЕЛЬСКИХ объектах ===
        
        var parentComponent = Components.GetInParent<PlayerController>();
        
        // === Создание и удаление ===
        
        // Создать новый компонент на этом объекте
        var newRenderer = Components.Create<ModelRenderer>();
        newRenderer.Model = Model.Load( "models/dev/box.vmdl" );
        
        // Удалить компонент
        var toRemove = Components.Get<ModelRenderer>();
        if ( toRemove != null )
        {
            toRemove.Destroy();
        }
    }
}
```

---

## Transform: Позиция, вращение, масштаб

Каждый GameObject имеет **Transform** — его положение в мире.

### Локальные vs Мировые координаты

```csharp
public sealed class TransformExample : Component
{
    protected override void OnUpdate()
    {
        // === МИРОВЫЕ координаты (World) ===
        // Позиция относительно НАЧАЛА МИРА (0, 0, 0)
        
        Vector3 worldPos = WorldPosition;       // Где объект в мире
        Rotation worldRot = WorldRotation;      // Вращение в мире
        Vector3 worldScale = WorldScale;        // Масштаб в мире
        
        // Установить мировую позицию
        WorldPosition = new Vector3( 100, 0, 50 );
        
        // === ЛОКАЛЬНЫЕ координаты (Local) ===
        // Позиция относительно РОДИТЕЛЬСКОГО объекта
        
        Vector3 localPos = LocalPosition;       // Смещение от родителя
        Rotation localRot = LocalRotation;      // Вращение относительно родителя
        Vector3 localScale = LocalScale;        // Масштаб относительно родителя
        
        // Установить локальную позицию (50 единиц правее родителя)
        LocalPosition = new Vector3( 0, 50, 0 );
    }
}
```

### Примеры перемещения

```csharp
public sealed class MovementExamples : Component
{
    [Property] public float Speed { get; set; } = 200f;
    
    protected override void OnUpdate()
    {
        // Пример 1: Движение вперёд (в направлении взгляда)
        WorldPosition += WorldRotation.Forward * Speed * Time.Delta;
        
        // Пример 2: Движение к точке
        Vector3 target = new Vector3( 500, 0, 0 );
        WorldPosition = Vector3.Lerp( WorldPosition, target, Time.Delta * 2f );
        
        // Пример 3: Движение по кругу
        float angle = Time.Now * 2f; // Скорость вращения
        float radius = 200f;
        WorldPosition = new Vector3(
            MathF.Cos( angle ) * radius,
            MathF.Sin( angle ) * radius,
            WorldPosition.z
        );
        
        // Пример 4: Движение вверх-вниз (синусоида)
        float baseHeight = 100f;
        float amplitude = 50f;
        WorldPosition = WorldPosition.WithZ( baseHeight + MathF.Sin( Time.Now * 3f ) * amplitude );
    }
}
```

### Примеры вращения

```csharp
public sealed class RotationExamples : Component
{
    [Property] public float RotateSpeed { get; set; } = 90f; // Градусов в секунду
    
    protected override void OnUpdate()
    {
        // Пример 1: Постоянное вращение вокруг оси Y (горизонтальное)
        WorldRotation *= Rotation.FromAxis( Vector3.Up, RotateSpeed * Time.Delta );
        
        // Пример 2: Смотреть на точку
        Vector3 targetPos = new Vector3( 500, 0, 100 );
        Vector3 direction = (targetPos - WorldPosition).Normal;
        WorldRotation = Rotation.LookAt( direction );
        
        // Пример 3: Плавный поворот к цели
        Rotation targetRotation = Rotation.LookAt( direction );
        WorldRotation = Rotation.Slerp( WorldRotation, targetRotation, Time.Delta * 5f );
        
        // Пример 4: Вращение из углов Эйлера
        Angles angles = WorldRotation.Angles();
        angles.yaw += RotateSpeed * Time.Delta;  // Горизонтальный поворот
        angles.pitch = 0;  // Без наклона вверх/вниз
        angles.roll = 0;   // Без крена
        WorldRotation = angles.ToRotation();
    }
}
```

---

## Практический пример: Собираемый предмет (Pickup)

Вот полный пример компонента для предмета, который можно подобрать:

```csharp
/// <summary>
/// Предмет, который игрок может подобрать.
/// Вращается и парит над землёй для привлечения внимания.
/// </summary>
[Title( "Подбираемый предмет" )]
[Category( "Игровая логика" )]
[Icon( "star" )]
public sealed class PickupItem : Component, Component.ITriggerListener
{
    /// <summary>Тип предмета</summary>
    [Property]
    public string ItemType { get; set; } = "health";
    
    /// <summary>Количество (например, +50 здоровья)</summary>
    [Property, Range( 1, 100 )]
    public int Amount { get; set; } = 25;
    
    /// <summary>Скорость вращения</summary>
    [Property]
    public float SpinSpeed { get; set; } = 90f;
    
    /// <summary>Амплитуда парения</summary>
    [Property]
    public float BobAmplitude { get; set; } = 10f;
    
    /// <summary>Скорость парения</summary>
    [Property]
    public float BobSpeed { get; set; } = 2f;
    
    /// <summary>Звук подбора</summary>
    [Property]
    public SoundEvent PickupSound { get; set; }
    
    private Vector3 startPosition;
    private bool isPickedUp;
    
    protected override void OnAwake()
    {
        startPosition = WorldPosition;
        isPickedUp = false;
    }
    
    protected override void OnUpdate()
    {
        if ( isPickedUp ) return;
        
        // Вращаем предмет
        WorldRotation *= Rotation.FromAxis( Vector3.Up, SpinSpeed * Time.Delta );
        
        // Плавное парение вверх-вниз
        float bob = MathF.Sin( Time.Now * BobSpeed ) * BobAmplitude;
        WorldPosition = startPosition + Vector3.Up * bob;
    }
    
    // ITriggerListener — вызывается когда другой объект входит в триггер
    void ITriggerListener.OnTriggerEnter( Collider other )
    {
        if ( isPickedUp ) return;
        
        // Проверяем, что это игрок
        var player = other.Components.GetInParent<PlayerController>();
        if ( player == null ) return;
        
        // Применяем эффект
        switch ( ItemType )
        {
            case "health":
                // player.Health += Amount;
                Log.Info( $"Игрок получил {Amount} здоровья!" );
                break;
            case "ammo":
                // player.Ammo += Amount;
                Log.Info( $"Игрок получил {Amount} патронов!" );
                break;
            case "armor":
                // player.Armor += Amount;
                Log.Info( $"Игрок получил {Amount} брони!" );
                break;
        }
        
        // Играем звук
        if ( PickupSound != null )
        {
            Sound.Play( PickupSound, WorldPosition );
        }
        
        // Удаляем предмет
        isPickedUp = true;
        GameObject.Destroy();
    }
    
    void ITriggerListener.OnTriggerExit( Collider other ) { }
}
```

---

## Практический пример: Простой контроллер игрока

```csharp
/// <summary>
/// Простой контроллер от первого лица.
/// Управляет движением и камерой.
/// </summary>
[Title( "Простой FPS контроллер" )]
[Category( "Игрок" )]
[Icon( "directions_run" )]
public sealed class SimpleFPSController : Component
{
    [Property, Group( "Движение" )]
    public float WalkSpeed { get; set; } = 200f;
    
    [Property, Group( "Движение" )]
    public float RunSpeed { get; set; } = 400f;
    
    [Property, Group( "Движение" )]
    public float JumpForce { get; set; } = 350f;
    
    [Property, Group( "Камера" )]
    public float MouseSensitivity { get; set; } = 1.0f;
    
    [Property, Group( "Ссылки" )]
    public GameObject CameraObject { get; set; }
    
    // Приватные переменные
    private CharacterController controller;
    private Angles eyeAngles;
    
    protected override void OnAwake()
    {
        // Получаем CharacterController на этом же объекте
        controller = Components.Get<CharacterController>();
    }
    
    protected override void OnUpdate()
    {
        // === КАМЕРА ===
        
        // Получаем движение мыши
        var look = Input.AnalogLook;
        
        // Обновляем углы камеры
        eyeAngles.yaw -= look.yaw * MouseSensitivity;
        eyeAngles.pitch += look.pitch * MouseSensitivity;
        eyeAngles.pitch = eyeAngles.pitch.Clamp( -89f, 89f ); // Ограничиваем наклон
        eyeAngles.roll = 0;
        
        // Применяем вращение к камере
        if ( CameraObject != null )
        {
            CameraObject.WorldRotation = eyeAngles.ToRotation();
        }
        
        // Поворачиваем тело только по горизонтали
        WorldRotation = Rotation.FromYaw( eyeAngles.yaw );
        
        // === ДВИЖЕНИЕ ===
        
        if ( controller == null ) return;
        
        // Определяем скорость (бег или ходьба)
        float speed = Input.Down( "Run" ) ? RunSpeed : WalkSpeed;
        
        // Получаем направление движения от WASD
        Vector3 moveDirection = Input.AnalogMove;
        
        // Преобразуем направление в мировые координаты
        Vector3 wishVelocity = WorldRotation * moveDirection * speed;
        
        // Если на земле
        if ( controller.IsOnGround )
        {
            // Применяем трение
            controller.ApplyFriction( 5.0f );
            
            // Ускоряем
            controller.Accelerate( wishVelocity );
            
            // Прыжок
            if ( Input.Pressed( "Jump" ) )
            {
                controller.Punch( Vector3.Up * JumpForce );
            }
        }
        else
        {
            // В воздухе — меньше контроля
            controller.Accelerate( wishVelocity * 0.1f );
            
            // Гравитация
            controller.Velocity += Scene.PhysicsWorld.Gravity * Time.Delta;
        }
        
        // Выполняем движение
        controller.Move();
    }
}
```

---

## Интерфейсы компонентов (Markers)

s&box предоставляет интерфейсы, которые ваши компоненты могут реализовать для получения уведомлений:

```csharp
// Получать уведомления о столкновениях
public sealed class MyComponent : Component, Component.ICollisionListener
{
    void ICollisionListener.OnCollisionStart( Collision collision )
    {
        Log.Info( $"Столкнулся с: {collision.Other.GameObject.Name}" );
        Log.Info( $"Скорость удара: {collision.Contact.Speed}" );
    }
    
    void ICollisionListener.OnCollisionUpdate( Collision collision ) { }
    void ICollisionListener.OnCollisionStop( CollisionStop collision ) { }
}

// Получать уведомления о триггерах (зонах)
public sealed class TriggerZone : Component, Component.ITriggerListener
{
    void ITriggerListener.OnTriggerEnter( Collider other )
    {
        Log.Info( $"{other.GameObject.Name} вошёл в зону!" );
    }
    
    void ITriggerListener.OnTriggerExit( Collider other )
    {
        Log.Info( $"{other.GameObject.Name} вышел из зоны!" );
    }
}

// Получать уведомления о сетевых событиях
public sealed class NetworkedComponent : Component, Component.INetworkListener
{
    void INetworkListener.OnConnected( Connection connection )
    {
        Log.Info( $"Игрок подключился: {connection.DisplayName}" );
    }
    
    void INetworkListener.OnDisconnected( Connection connection )
    {
        Log.Info( $"Игрок отключился: {connection.DisplayName}" );
    }
}
```

---

## Следующий шаг

Перейдите к разделу **[Система сцен](../03-Scene-System/README.md)** чтобы узнать, как создавать и управлять сценами.
