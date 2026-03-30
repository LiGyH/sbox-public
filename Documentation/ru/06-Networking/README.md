# 6. Сетевая игра (Networking)

## Что такое сетевая игра в s&box?

В s&box **мультиплеер встроен в движок**. Вам не нужно писать сетевой код с нуля — движок берёт на себя синхронизацию, передачу данных и управление соединениями.

### Основные понятия

| Термин | Описание |
|--------|----------|
| **Host (Хост)** | Игрок, который создал сервер. У него полный контроль |
| **Client (Клиент)** | Игрок, подключившийся к серверу |
| **Owner (Владелец)** | Игрок, который «владеет» объектом (может управлять) |
| **Proxy (Прокси)** | Копия чужого объекта на вашем клиенте (только чтение) |
| **RPC** | Remote Procedure Call — удалённый вызов метода |
| **[Sync]** | Автоматическая синхронизация свойства по сети |

### Как это работает?

```
Хост (Сервер)           Клиент 1              Клиент 2
     │                      │                      │
     │ ← Подключается ──── │                      │
     │ ← Подключается ──── │ ──────────────────── │
     │                      │                      │
     │ ── Синхронизация ──→ │                      │
     │ ── Синхронизация ──→ │ ──────────────────→  │
     │                      │                      │
     │ ← RPC (запрос) ──── │                      │
     │ ── RPC (ответ) ───→ │                      │
```

---

## Базовая настройка сетевого объекта

### Шаг 1: Создайте компонент

```csharp
public sealed class NetworkedPlayer : Component, Component.INetworkListener
{
    // [Sync] — это свойство автоматически синхронизируется по сети!
    [Sync]
    public string PlayerName { get; set; } = "Игрок";
    
    [Sync]
    public float Health { get; set; } = 100f;
    
    [Sync]
    public int Score { get; set; }
    
    [Sync]
    public Color TeamColor { get; set; } = Color.White;
    
    // Это свойство НЕ синхронизируется (нет [Sync])
    private float localTimer;
    
    protected override void OnUpdate()
    {
        // Проверяем, является ли этот объект нашим (не прокси)
        if ( IsProxy ) return; // Чужой объект — не управляем
        
        // Только владелец может двигать объект
        HandleMovement();
    }
    
    void HandleMovement()
    {
        var moveDir = Input.AnalogMove;
        WorldPosition += WorldRotation * moveDir * 200f * Time.Delta;
    }
    
    // Вызывается когда этот объект подключается к сети
    void INetworkListener.OnConnected( Connection connection )
    {
        Log.Info( $"Объект подключён к сети. Владелец: {connection.DisplayName}" );
    }
    
    void INetworkListener.OnDisconnected( Connection connection )
    {
        Log.Info( $"Владелец отключился: {connection.DisplayName}" );
    }
}
```

### Шаг 2: Спавн сетевого объекта

```csharp
public sealed class GameManager : Component, Component.INetworkListener
{
    [Property]
    public PrefabFile PlayerPrefab { get; set; }
    
    // Вызывается когда новый игрок подключается
    void INetworkListener.OnConnected( Connection connection )
    {
        // Спавним объект для нового игрока
        var playerObj = SceneUtility.Instantiate( PlayerPrefab, GetSpawnPosition(), Rotation.Identity );
        
        // Регистрируем объект в сети и назначаем владельца
        playerObj.NetworkSpawn( connection );
        
        Log.Info( $"Игрок {connection.DisplayName} заспавнен!" );
    }
    
    Vector3 GetSpawnPosition()
    {
        // Находим случайную точку спавна
        var spawnPoints = Scene.GetAllComponents<SpawnPoint>();
        var point = spawnPoints.ElementAtOrDefault( Random.Shared.Next( spawnPoints.Count() ) );
        return point?.WorldPosition ?? Vector3.Zero;
    }
}
```

---

## Атрибут [Sync] — синхронизация свойств

`[Sync]` автоматически синхронизирует значение свойства по сети:

```csharp
public sealed class SyncExample : Component
{
    // === ПРОСТЫЕ ТИПЫ ===
    
    [Sync] public int Score { get; set; }
    [Sync] public float Health { get; set; }
    [Sync] public bool IsAlive { get; set; }
    [Sync] public string Name { get; set; }
    
    // === ТИПЫ s&box ===
    
    [Sync] public Vector3 TargetPosition { get; set; }
    [Sync] public Rotation LookRotation { get; set; }
    [Sync] public Color TeamColor { get; set; }
    [Sync] public Angles EyeAngles { get; set; }
    
    // === ВАЖНО: Только владелец меняет [Sync] свойства ===
    
    protected override void OnUpdate()
    {
        // Только если мы владелец этого объекта
        if ( IsProxy ) return;
        
        // Изменяем свойство — оно автоматически отправится всем клиентам
        Health -= 1f * Time.Delta;
        TargetPosition = WorldPosition;
        LookRotation = WorldRotation;
    }
}
```

### Правила [Sync]:
1. **Только владелец** может изменять `[Sync]` свойства
2. Изменения **автоматически отправляются** всем клиентам
3. Поддерживаемые типы: `int`, `float`, `bool`, `string`, `Vector3`, `Rotation`, `Color`, `Angles` и другие
4. **Не используйте** для данных, которые меняются каждый кадр (слишком много трафика)

---

## RPC — удалённый вызов процедур

RPC позволяет вызывать методы на **других компьютерах**:

### [Broadcast] — вызов на ВСЕХ клиентах

```csharp
public sealed class BroadcastExample : Component
{
    [Sync] public float Health { get; set; } = 100f;
    
    // Этот метод выполнится на ВСЕХ подключённых клиентах
    [Broadcast]
    public void TakeDamage( float damage )
    {
        Health -= damage;
        
        Log.Info( $"Получено {damage} урона! Здоровье: {Health}" );
        
        if ( Health <= 0 )
        {
            Die();
        }
    }
    
    // Выполнится на ВСЕХ — например, визуальный эффект взрыва
    [Broadcast]
    public void PlayExplosionEffect( Vector3 position )
    {
        // Каждый клиент увидит эффект
        Log.Info( $"Взрыв на позиции: {position}" );
        // Спавним частицы, звук и т.д.
    }
    
    void Die()
    {
        Log.Info( "Игрок погиб!" );
    }
}
```

### [Authority] — только владелец/хост может вызвать

```csharp
public sealed class AuthorityExample : Component
{
    [Sync] public float Health { get; set; } = 100f;
    [Sync] public int Score { get; set; }
    
    // Только ХОСТ может вызвать этот метод (валидация на сервере)
    [Broadcast( NetPermission.HostOnly )]
    public void AddScore( int amount )
    {
        Score += amount;
        Log.Info( $"Счёт: {Score}" );
    }
    
    // Только ВЛАДЕЛЕЦ объекта может вызвать
    [Broadcast( NetPermission.OwnerOnly )]
    public void RequestHeal( float amount )
    {
        Health = MathF.Min( Health + amount, 100f );
        Log.Info( $"Исцелён на {amount}. Здоровье: {Health}" );
    }
}
```

### Пример: Система чата

```csharp
public sealed class ChatSystem : Component
{
    // Список сообщений (только на клиенте, не синхронизируется)
    public List<ChatMessage> Messages { get; set; } = new();
    
    /// <summary>
    /// Отправить сообщение в чат. Вызывается на клиенте,
    /// выполняется на всех клиентах.
    /// </summary>
    [Broadcast]
    public void SendMessage( string author, string text )
    {
        Messages.Add( new ChatMessage
        {
            Author = author,
            Text = text,
            Time = DateTime.Now
        });
        
        // Ограничиваем историю
        while ( Messages.Count > 100 )
            Messages.RemoveAt( 0 );
    }
    
    // Системное сообщение (только хост может отправить)
    [Broadcast( NetPermission.HostOnly )]
    public void SendSystemMessage( string text )
    {
        Messages.Add( new ChatMessage
        {
            Author = "СИСТЕМА",
            Text = text,
            Time = DateTime.Now,
            IsSystem = true
        });
    }
    
    public class ChatMessage
    {
        public string Author { get; set; }
        public string Text { get; set; }
        public DateTime Time { get; set; }
        public bool IsSystem { get; set; }
    }
}
```

---

## Connection — работа с подключениями

```csharp
public sealed class ConnectionExample : Component
{
    protected override void OnUpdate()
    {
        // Проверяем, является ли текущий клиент хостом
        if ( Networking.IsHost )
        {
            // Только хост видит это
        }
        
        // Наше подключение
        Connection localConnection = Connection.Local;
        
        // Все подключения
        foreach ( var conn in Connection.All )
        {
            Log.Info( $"Игрок: {conn.DisplayName}, ID: {conn.Id}" );
        }
    }
    
    // Получить информацию о подключении
    void PrintConnectionInfo( Connection conn )
    {
        Log.Info( $"Имя: {conn.DisplayName}" );
        Log.Info( $"SteamID: {conn.SteamId}" );
        Log.Info( $"ID: {conn.Id}" );
    }
}
```

---

## Полный пример: Сетевой контроллер игрока

```csharp
/// <summary>
/// Полный сетевой контроллер от первого лица.
/// Синхронизирует движение, камеру и действия по сети.
/// </summary>
[Title( "Сетевой FPS контроллер" )]
[Category( "Игрок" )]
[Icon( "person" )]
public sealed class NetworkFPSController : Component, Component.INetworkListener
{
    // === НАСТРОЙКИ (Property) ===
    [Property, Group( "Движение" )] public float WalkSpeed { get; set; } = 200f;
    [Property, Group( "Движение" )] public float RunSpeed { get; set; } = 400f;
    [Property, Group( "Движение" )] public float JumpForce { get; set; } = 350f;
    [Property, Group( "Ссылки" )] public GameObject CameraObject { get; set; }
    [Property, Group( "Ссылки" )] public ModelRenderer BodyRenderer { get; set; }
    
    // === СИНХРОНИЗИРУЕМЫЕ СВОЙСТВА ===
    [Sync] public float Health { get; set; } = 100f;
    [Sync] public string PlayerName { get; set; }
    [Sync] public Angles EyeAngles { get; set; }
    [Sync] public bool IsCrouching { get; set; }
    [Sync] public bool IsRunning { get; set; }
    
    private CharacterController controller;
    
    protected override void OnAwake()
    {
        controller = Components.Get<CharacterController>();
    }
    
    protected override void OnUpdate()
    {
        // === Только владелец управляет объектом ===
        if ( !IsProxy )
        {
            HandleCamera();
            HandleMovement();
            HandleActions();
        }
        
        // === Все клиенты обновляют визуал ===
        UpdateVisuals();
    }
    
    void HandleCamera()
    {
        var look = Input.AnalogLook;
        var angles = EyeAngles;
        angles.yaw -= look.yaw;
        angles.pitch += look.pitch;
        angles.pitch = angles.pitch.Clamp( -89f, 89f );
        angles.roll = 0;
        EyeAngles = angles;
        
        if ( CameraObject != null )
        {
            CameraObject.WorldRotation = EyeAngles.ToRotation();
        }
        
        WorldRotation = Rotation.FromYaw( EyeAngles.yaw );
    }
    
    void HandleMovement()
    {
        if ( controller == null ) return;
        
        IsRunning = Input.Down( "Run" );
        IsCrouching = Input.Down( "Duck" );
        
        float speed = IsRunning ? RunSpeed : WalkSpeed;
        if ( IsCrouching ) speed *= 0.5f;
        
        var moveDir = Input.AnalogMove;
        var wishVelocity = WorldRotation * moveDir * speed;
        
        if ( controller.IsOnGround )
        {
            controller.ApplyFriction( 5f );
            controller.Accelerate( wishVelocity );
            
            if ( Input.Pressed( "Jump" ) )
            {
                controller.Punch( Vector3.Up * JumpForce );
            }
        }
        else
        {
            controller.Accelerate( wishVelocity * 0.1f );
            controller.Velocity += Scene.PhysicsWorld.Gravity * Time.Delta;
        }
        
        controller.Move();
    }
    
    void HandleActions()
    {
        if ( Input.Pressed( "Attack1" ) )
        {
            Shoot();
        }
    }
    
    void Shoot()
    {
        Vector3 start = CameraObject?.WorldPosition ?? WorldPosition;
        Vector3 end = start + EyeAngles.ToRotation().Forward * 5000f;
        
        var trace = Scene.Trace
            .Ray( start, end )
            .IgnoreGameObject( GameObject )
            .Run();
        
        if ( trace.Hit )
        {
            var targetHealth = trace.GameObject?.Components
                .GetInParent<NetworkFPSController>();
            
            if ( targetHealth != null )
            {
                // Наносим урон через RPC
                targetHealth.TakeDamage( 25f, PlayerName );
            }
        }
        
        // Визуальный эффект выстрела на всех клиентах
        BroadcastShootEffect( start, trace.HitPosition ?? end );
    }
    
    [Broadcast]
    public void TakeDamage( float damage, string attackerName )
    {
        Health -= damage;
        Log.Info( $"{PlayerName} получил {damage} урона от {attackerName}" );
        
        if ( Health <= 0 )
        {
            BroadcastDeath( attackerName );
        }
    }
    
    [Broadcast]
    void BroadcastShootEffect( Vector3 from, Vector3 to )
    {
        // Все клиенты видят трейсер пули
        // Здесь можно спавнить частицы, звук и т.д.
    }
    
    [Broadcast]
    void BroadcastDeath( string killerName )
    {
        Log.Info( $"{PlayerName} убит {killerName}!" );
        // Анимация смерти, респавн и т.д.
    }
    
    void UpdateVisuals()
    {
        // Скрываем модель для владельца (FPS вид)
        if ( BodyRenderer != null )
        {
            BodyRenderer.Enabled = IsProxy; // Видна только для других
        }
    }
    
    void INetworkListener.OnConnected( Connection connection )
    {
        PlayerName = connection.DisplayName;
    }
    
    void INetworkListener.OnDisconnected( Connection connection ) { }
}
```

---

## Передача владения объектом

```csharp
public sealed class OwnershipExample : Component
{
    public void PickupItem( GameObject item, Connection newOwner )
    {
        // Передаём владение предметом другому игроку
        item.Network.SetOwnerTransfer( OwnerTransfer.Takeover );
        item.Network.AssignOwnership( newOwner );
    }
    
    public void DropItem( GameObject item )
    {
        // Убираем владение (объект становится "ничей")
        item.Network.DropOwnership();
    }
}
```

---

## Проверки сети

```csharp
public sealed class NetworkChecks : Component
{
    protected override void OnUpdate()
    {
        // Мы хост?
        bool isHost = Networking.IsHost;
        
        // Этот объект — прокси (чужой)?
        bool proxy = IsProxy;
        
        // Мы владелец этого объекта?
        bool owner = !IsProxy; // Если не прокси — мы владелец
        
        // Объект зарегистрирован в сети?
        bool networked = Network.Active;
        
        // Количество подключений
        int playerCount = Connection.All.Count();
    }
}
```

---

## Частые ошибки сетевого кода

### ❌ Ошибка 1: Управление прокси
```csharp
// ❌ Все клиенты двигают объект — хаос!
protected override void OnUpdate()
{
    WorldPosition += Input.AnalogMove * Speed * Time.Delta;
}

// ✅ Только владелец управляет
protected override void OnUpdate()
{
    if ( IsProxy ) return; // Чужой объект — пропускаем
    WorldPosition += Input.AnalogMove * Speed * Time.Delta;
}
```

### ❌ Ошибка 2: Забыли NetworkSpawn
```csharp
// ❌ Объект не синхронизируется — забыли зарегистрировать
var obj = new GameObject();
// obj.NetworkSpawn(); ← ЗАБЫЛИ!

// ✅ Регистрируем после создания
var obj = new GameObject();
obj.NetworkSpawn( connection );
```

### ❌ Ошибка 3: Изменение [Sync] свойства на прокси
```csharp
// ❌ Прокси не может менять [Sync] свойства
[Sync] public float Health { get; set; }

protected override void OnUpdate()
{
    Health -= Time.Delta; // НЕ РАБОТАЕТ на прокси!
}

// ✅ Проверяем владельца
protected override void OnUpdate()
{
    if ( IsProxy ) return;
    Health -= Time.Delta; // ОК — мы владелец
}
```

---

## Следующий шаг

Перейдите к разделу **[Физика и столкновения](../07-Physics-Collision/README.md)** для изучения физической системы.
