# 7. Физика и столкновения

## Физический мир s&box

s&box использует физический движок для реалистичной симуляции:
- **Гравитация** — объекты падают
- **Столкновения** — объекты не проходят друг через друга
- **Трение и отскоки** — реалистичное поведение поверхностей
- **Шарниры (Joints)** — связи между объектами

---

## Rigidbody — физическое тело

`Rigidbody` делает объект **физическим**: на него действует гравитация, он сталкивается с другими объектами и реагирует на силы.

```csharp
public sealed class PhysicsExample : Component
{
    protected override void OnAwake()
    {
        // Получаем Rigidbody
        var body = Components.Get<Rigidbody>();
        if ( body == null ) return;
        
        // Включаем/выключаем гравитацию
        body.Gravity = true;
        
        // Задаём начальную скорость
        body.Velocity = Vector3.Forward * 500f;
        
        // Угловая скорость (вращение)
        body.AngularVelocity = new Vector3( 0, 0, 180 ); // Вращаем вокруг Z
    }
    
    // Толкнуть объект
    public void Push( Vector3 force )
    {
        var body = Components.Get<Rigidbody>();
        if ( body == null ) return;
        
        // ApplyForce — плавная сила (как ветер)
        body.ApplyForce( force );
        
        // ApplyForceAt — сила в точке (создаёт вращение)
        body.ApplyForceAt( WorldPosition + Vector3.Up * 50, force );
    }
    
    // Взрыв — толкает все объекты в радиусе
    public void Explode( Vector3 center, float radius, float force )
    {
        foreach ( var body in Scene.GetAllComponents<Rigidbody>() )
        {
            float dist = body.WorldPosition.Distance( center );
            if ( dist > radius ) continue;
            
            // Сила уменьшается с расстоянием
            float strength = (1f - dist / radius) * force;
            Vector3 direction = (body.WorldPosition - center).Normal;
            
            body.ApplyForce( direction * strength );
        }
    }
}
```

---

## Коллайдеры (Colliders)

Коллайдеры определяют **форму** объекта для столкновений:

| Коллайдер | Описание | Когда использовать |
|-----------|----------|--------------------|
| `BoxCollider` | Прямоугольник | Ящики, стены, двери |
| `SphereCollider` | Сфера | Мячи, бомбы |
| `CapsuleCollider` | Капсула | Игроки, NPC |
| `ModelCollider` | Форма модели | Сложные объекты |
| `PlaneCollider` | Бесконечная плоскость | Земля, пол |
| `HullCollider` | Выпуклая оболочка | Камни, предметы |

### Создание коллайдеров из кода

```csharp
public sealed class ColliderSetup : Component
{
    protected override void OnAwake()
    {
        // Box (прямоугольник)
        var box = Components.Create<BoxCollider>();
        box.Scale = new Vector3( 50, 50, 50 ); // Размер 50x50x50
        box.Center = Vector3.Zero;              // В центре объекта
        
        // Sphere (сфера)
        var sphere = Components.Create<SphereCollider>();
        sphere.Radius = 25f;
        sphere.Center = Vector3.Zero;
        
        // Capsule (капсула) — для персонажей
        var capsule = Components.Create<CapsuleCollider>();
        capsule.Radius = 16f;
        capsule.Start = Vector3.Zero;
        capsule.End = Vector3.Up * 72f; // Высота 72 единицы
    }
}
```

### Триггеры

Коллайдер может быть **триггером** — зоной, которая не блокирует движение, но **обнаруживает** объекты внутри:

```csharp
public sealed class TriggerSetup : Component
{
    protected override void OnAwake()
    {
        var collider = Components.Get<BoxCollider>();
        if ( collider != null )
        {
            collider.IsTrigger = true; // Делаем триггером
        }
    }
}
```

---

## Обнаружение столкновений

### ICollisionListener — физические столкновения

```csharp
/// <summary>
/// Компонент, который реагирует на физические столкновения.
/// </summary>
public sealed class CollisionHandler : Component, Component.ICollisionListener
{
    [Property] public float BounceDamage { get; set; } = 10f;
    [Property] public SoundEvent HitSound { get; set; }
    
    // Вызывается когда объект НАЧИНАЕТ сталкиваться с другим
    void ICollisionListener.OnCollisionStart( Collision collision )
    {
        // Информация о столкновении
        var otherObject = collision.Other.GameObject;
        var contactPoint = collision.Contact.Point;     // Точка касания
        var contactNormal = collision.Contact.Normal;   // Нормаль поверхности
        var impactSpeed = collision.Contact.Speed;      // Скорость удара
        
        Log.Info( $"Столкнулся с: {otherObject.Name}" );
        Log.Info( $"Скорость удара: {impactSpeed}" );
        
        // Наносим урон при сильном ударе
        if ( impactSpeed > 200f )
        {
            var health = otherObject.Components.Get<HealthComponent>();
            if ( health != null )
            {
                health.TakeDamage( impactSpeed * 0.1f );
            }
        }
        
        // Звук удара
        if ( HitSound != null && impactSpeed > 50f )
        {
            Sound.Play( HitSound, contactPoint );
        }
    }
    
    // Вызывается КАЖДЫЙ КАДР пока объекты соприкасаются
    void ICollisionListener.OnCollisionUpdate( Collision collision )
    {
        // Можно использовать для постоянного эффекта (трение, искры)
    }
    
    // Вызывается когда объекты ПРЕКРАЩАЮТ соприкасаться
    void ICollisionListener.OnCollisionStop( CollisionStop collision )
    {
        Log.Info( $"Столкновение с {collision.Other.GameObject.Name} завершено" );
    }
}
```

### ITriggerListener — вход/выход из зоны

```csharp
/// <summary>
/// Зона, которая обнаруживает объекты внутри.
/// Коллайдер должен быть с IsTrigger = true!
/// </summary>
public sealed class TriggerZone : Component, Component.ITriggerListener
{
    [Property] public string ZoneName { get; set; } = "Зона";
    
    // Список объектов внутри зоны
    private List<GameObject> objectsInside = new();
    
    // Вызывается когда объект ВХОДИТ в зону
    void ITriggerListener.OnTriggerEnter( Collider other )
    {
        objectsInside.Add( other.GameObject );
        Log.Info( $"{other.GameObject.Name} вошёл в {ZoneName}" );
        
        // Пример: нанести урон при входе
        var health = other.Components.GetInParent<HealthComponent>();
        if ( health != null )
        {
            health.TakeDamage( 50f );
        }
    }
    
    // Вызывается когда объект ПОКИДАЕТ зону
    void ITriggerListener.OnTriggerExit( Collider other )
    {
        objectsInside.Remove( other.GameObject );
        Log.Info( $"{other.GameObject.Name} покинул {ZoneName}" );
    }
    
    // Сколько объектов сейчас внутри
    public int CountInside => objectsInside.Count;
}
```

---

## Трассировка лучей (Raycast) — подробно

### Все виды трассировки

```csharp
public sealed class TraceExamples : Component
{
    protected override void OnUpdate()
    {
        // === 1. ПРОСТОЙ ЛУЧ (Ray) ===
        // Тонкая линия из точки A в точку B
        var rayResult = Scene.Trace
            .Ray( WorldPosition, WorldPosition + Vector3.Forward * 1000 )
            .Run();
        
        // === 2. СФЕРИЧЕСКИЙ ЛУЧ (Sphere) ===
        // Как катящийся мяч (толстый луч)
        var sphereResult = Scene.Trace
            .Sphere( 16f, WorldPosition, WorldPosition + Vector3.Forward * 1000 )
            .Run();
        
        // === 3. ПРЯМОУГОЛЬНЫЙ ЛУЧ (Box) ===
        // Как двигающийся ящик
        var boxResult = Scene.Trace
            .Box( new Vector3( 32, 32, 64 ), WorldPosition, WorldPosition + Vector3.Down * 200 )
            .Run();
        
        // === 4. КАПСУЛЬНЫЙ ЛУЧ ===
        var capsuleResult = Scene.Trace
            .Capsule( new Capsule( Vector3.Zero, Vector3.Up * 72, 16f ) )
            .Run();
    }
}
```

### Фильтры трассировки

```csharp
public sealed class TraceFilters : Component
{
    protected override void OnUpdate()
    {
        var trace = Scene.Trace
            .Ray( WorldPosition, WorldPosition + Vector3.Forward * 5000 );
        
        // === ФИЛЬТРЫ ПО ТЕГАМ ===
        
        // Только объекты с тегом "solid"
        trace = trace.WithTag( "solid" );
        
        // Объекты с ЛЮБЫМ из указанных тегов
        trace = trace.WithAnyTags( "solid", "player", "npc" );
        
        // Объекты со ВСЕМИ указанными тегами
        trace = trace.WithAllTags( "solid", "breakable" );
        
        // Исключить объекты с тегами
        trace = trace.WithoutTags( "trigger", "invisible" );
        
        // === ФИЛЬТРЫ ПО ОБЪЕКТАМ ===
        
        // Игнорировать конкретный объект
        trace = trace.IgnoreGameObject( GameObject );
        
        // === ВЫПОЛНЕНИЕ ===
        
        var result = trace.Run();
        
        if ( result.Hit )
        {
            ProcessHit( result );
        }
    }
    
    void ProcessHit( SceneTraceResult result )
    {
        // Что мы получаем из результата:
        
        bool hit = result.Hit;                    // Попали ли куда-то
        Vector3 hitPos = result.HitPosition;      // Точка попадания
        Vector3 normal = result.Normal;           // Нормаль поверхности
        float distance = result.Distance;         // Расстояние до попадания
        float fraction = result.Fraction;         // 0-1, доля пройденного пути
        GameObject hitObj = result.GameObject;    // Объект, в который попали
        
        // Пример использования нормали: размещение декали (следа пули)
        // на поверхности — используем HitPosition и Normal
        
        Log.Info( $"Попал в: {hitObj?.Name}" );
        Log.Info( $"Позиция: {hitPos}" );
        Log.Info( $"Расстояние: {distance}" );
    }
}
```

---

## CharacterController — контроллер персонажа

`CharacterController` — специальный компонент для управления **персонажами**. Он обрабатывает столкновения без Rigidbody:

```csharp
/// <summary>
/// Полноценный контроллер персонажа с бегом, прыжком и приседанием.
/// </summary>
public sealed class PlayerMovement : Component
{
    // === НАСТРОЙКИ ===
    [Property, Group( "Скорости" )] public float WalkSpeed { get; set; } = 200f;
    [Property, Group( "Скорости" )] public float RunSpeed { get; set; } = 400f;
    [Property, Group( "Скорости" )] public float CrouchSpeed { get; set; } = 100f;
    [Property, Group( "Физика" )] public float JumpForce { get; set; } = 350f;
    [Property, Group( "Физика" )] public float Gravity { get; set; } = 800f;
    [Property, Group( "Физика" )] public float GroundFriction { get; set; } = 5f;
    [Property, Group( "Физика" )] public float AirControl { get; set; } = 0.1f;
    
    private CharacterController cc;
    
    protected override void OnAwake()
    {
        cc = Components.Get<CharacterController>();
    }
    
    protected override void OnUpdate()
    {
        if ( cc == null ) return;
        
        // Определяем скорость
        float speed = WalkSpeed;
        if ( Input.Down( "Run" ) ) speed = RunSpeed;
        if ( Input.Down( "Duck" ) ) speed = CrouchSpeed;
        
        // Направление движения
        Vector3 wishDir = Input.AnalogMove;
        Vector3 wishVelocity = WorldRotation * wishDir * speed;
        
        if ( cc.IsOnGround )
        {
            // === НА ЗЕМЛЕ ===
            cc.ApplyFriction( GroundFriction );
            cc.Accelerate( wishVelocity );
            
            // Прыжок
            if ( Input.Pressed( "Jump" ) )
            {
                cc.Punch( Vector3.Up * JumpForce );
            }
        }
        else
        {
            // === В ВОЗДУХЕ ===
            cc.Accelerate( wishVelocity * AirControl );
            
            // Гравитация
            cc.Velocity -= Vector3.Up * Gravity * Time.Delta;
        }
        
        // Выполняем движение (с учётом столкновений)
        cc.Move();
        
        // Ограничиваем вертикальную скорость
        if ( cc.IsOnGround && cc.Velocity.z < 0 )
        {
            cc.Velocity = cc.Velocity.WithZ( 0 );
        }
    }
    
    // Проверка: можно ли прыгнуть?
    bool CanJump()
    {
        return cc.IsOnGround;
    }
    
    // Проверка: на чём стоим?
    void CheckGround()
    {
        if ( cc.IsOnGround && cc.GroundObject != null )
        {
            Log.Info( $"Стоим на: {cc.GroundObject.Name}" );
        }
    }
}
```

---

## Шарниры (Joints)

Шарниры связывают два объекта вместе:

| Шарнир | Описание | Пример |
|--------|----------|--------|
| `FixedJoint` | Жёсткая связь | Приклеенные объекты |
| `HingeJoint` | Петля (одна ось вращения) | Дверь, крышка |
| `BallJoint` | Шар (вращение во все стороны) | Плечевой сустав |
| `SpringJoint` | Пружина | Подвеска машины |
| `SliderJoint` | Ползунок (движение по оси) | Выдвижной ящик |

### Пример: Дверь на петлях

```csharp
public sealed class HingeDoor : Component
{
    [Property] public float OpenAngle { get; set; } = 90f;
    [Property] public float OpenSpeed { get; set; } = 2f;
    
    private HingeJoint hinge;
    private bool isOpen;
    
    protected override void OnAwake()
    {
        hinge = Components.Get<HingeJoint>();
    }
    
    public void Toggle()
    {
        isOpen = !isOpen;
        // Управление через физику шарнира
    }
}
```

---

## Практический пример: Система урона

```csharp
/// <summary>
/// Компонент здоровья. Можно добавить к любому объекту,
/// чтобы он мог получать и наносить урон.
/// </summary>
[Title( "Здоровье" )]
[Category( "Игровая логика" )]
[Icon( "favorite" )]
public sealed class HealthComponent : Component
{
    [Property, Sync] public float MaxHealth { get; set; } = 100f;
    [Property, Sync] public float Health { get; set; } = 100f;
    [Property] public bool DestroyOnDeath { get; set; } = true;
    
    public bool IsAlive => Health > 0;
    public float HealthPercent => Health / MaxHealth;
    
    /// <summary>
    /// Нанести урон. Вызывайте с любого клиента.
    /// </summary>
    [Broadcast]
    public void TakeDamage( float damage )
    {
        if ( !IsAlive ) return;
        
        Health -= damage;
        Health = MathF.Max( Health, 0 );
        
        Log.Info( $"{GameObject.Name} получил {damage} урона. Здоровье: {Health}/{MaxHealth}" );
        
        if ( Health <= 0 )
        {
            OnDeath();
        }
    }
    
    /// <summary>
    /// Исцелить.
    /// </summary>
    [Broadcast]
    public void Heal( float amount )
    {
        if ( !IsAlive ) return;
        
        Health += amount;
        Health = MathF.Min( Health, MaxHealth );
        
        Log.Info( $"{GameObject.Name} исцелён на {amount}. Здоровье: {Health}/{MaxHealth}" );
    }
    
    void OnDeath()
    {
        Log.Info( $"{GameObject.Name} погиб!" );
        
        if ( DestroyOnDeath )
        {
            GameObject.Destroy();
        }
    }
}
```

---

## Следующий шаг

Перейдите к разделу **[Система ввода (Input)](../08-Input-System/README.md)** для изучения управления.
