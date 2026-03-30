# DarkRP: Шаг 2 — Компонент игрока

## DarkRPPlayer.cs — Главный компонент игрока

Этот компонент содержит **все данные игрока**: профессию, деньги, здоровье и т.д.

```csharp
// code/Player/DarkRPPlayer.cs

using Sandbox;
using System;

/// <summary>
/// Главный компонент игрока DarkRP.
/// Содержит все синхронизируемые данные: профессию, деньги, здоровье.
/// </summary>
[Title( "DarkRP Игрок" )]
[Category( "DarkRP" )]
[Icon( "person" )]
public sealed class DarkRPPlayer : Component, Component.INetworkListener
{
    // === СЕТЕВЫЕ СВОЙСТВА (синхронизируются автоматически) ===
    
    /// <summary>Имя игрока</summary>
    [Sync] public string PlayerName { get; set; } = "Игрок";
    
    /// <summary>Текущая профессия (название)</summary>
    [Sync] public string JobName { get; set; } = "Гражданин";
    
    /// <summary>Цвет профессии</summary>
    [Sync] public Color JobColor { get; set; } = Color.White;
    
    /// <summary>Деньги игрока</summary>
    [Sync] public int Money { get; set; } = 500;
    
    /// <summary>Здоровье</summary>
    [Sync] public float Health { get; set; } = 100f;
    
    /// <summary>Максимальное здоровье</summary>
    [Sync] public float MaxHealth { get; set; } = 100f;
    
    /// <summary>Броня</summary>
    [Sync] public int Armor { get; set; } = 0;
    
    /// <summary>Жив ли игрок</summary>
    [Sync] public bool IsAlive { get; set; } = true;
    
    /// <summary>Разыскивается ли полицией</summary>
    [Sync] public bool IsWanted { get; set; } = false;
    
    /// <summary>Углы камеры (для синхронизации обзора)</summary>
    [Sync] public Angles EyeAngles { get; set; }
    
    // === НАСТРОЙКИ (из редактора) ===
    
    [Property] public GameObject CameraObject { get; set; }
    [Property] public SkinnedModelRenderer BodyRenderer { get; set; }
    [Property] public float WalkSpeed { get; set; } = 200f;
    [Property] public float RunSpeed { get; set; } = 350f;
    
    // === ПРИВАТНЫЕ ПЕРЕМЕННЫЕ ===
    
    private CharacterController cc;
    private TimeSince lastSalary;
    private JobDefinition currentJob;
    
    // === ССЫЛКИ НА СИСТЕМЫ ===
    
    /// <summary>Является ли этот игрок локальным (наш клиент)</summary>
    public bool IsLocal => !IsProxy;
    
    /// <summary>Процент здоровья (0-1)</summary>
    public float HealthPercent => Health / MaxHealth;
    
    // ============================
    // ЖИЗНЕННЫЙ ЦИКЛ
    // ============================
    
    protected override void OnAwake()
    {
        cc = Components.Get<CharacterController>();
    }
    
    protected override void OnUpdate()
    {
        // Только владелец управляет персонажем
        if ( !IsProxy )
        {
            HandleCamera();
            HandleMovement();
            HandleInteraction();
            HandleSalary();
        }
        
        // Все клиенты обновляют визуал
        UpdateVisuals();
    }
    
    // ============================
    // КАМЕРА
    // ============================
    
    void HandleCamera()
    {
        var look = Input.AnalogLook;
        var angles = EyeAngles;
        angles.yaw -= look.yaw;
        angles.pitch += look.pitch;
        angles.pitch = angles.pitch.Clamp( -89f, 89f );
        angles.roll = 0;
        EyeAngles = angles;
        
        // Камера — полное вращение
        if ( CameraObject != null )
        {
            CameraObject.WorldRotation = EyeAngles.ToRotation();
            CameraObject.WorldPosition = WorldPosition + Vector3.Up * 64;
        }
        
        // Тело — только горизонтальный поворот
        WorldRotation = Rotation.FromYaw( EyeAngles.yaw );
    }
    
    // ============================
    // ДВИЖЕНИЕ
    // ============================
    
    void HandleMovement()
    {
        if ( cc == null ) return;
        if ( !IsAlive ) return;
        
        float speed = Input.Down( "Run" ) ? RunSpeed : WalkSpeed;
        var wishVel = WorldRotation * Input.AnalogMove * speed;
        
        if ( cc.IsOnGround )
        {
            cc.ApplyFriction( 5f );
            cc.Accelerate( wishVel );
            
            if ( Input.Pressed( "Jump" ) )
            {
                cc.Punch( Vector3.Up * 300f );
            }
        }
        else
        {
            cc.Accelerate( wishVel * 0.1f );
            cc.Velocity += Vector3.Down * 800f * Time.Delta;
        }
        
        cc.Move();
    }
    
    // ============================
    // ВЗАИМОДЕЙСТВИЕ (E)
    // ============================
    
    void HandleInteraction()
    {
        if ( !Input.Pressed( "Use" ) ) return;
        
        // Луч из глаз вперёд на 200 единиц
        var eyePos = CameraObject?.WorldPosition ?? WorldPosition + Vector3.Up * 64;
        var eyeDir = EyeAngles.ToRotation().Forward;
        
        var trace = Scene.Trace
            .Ray( eyePos, eyePos + eyeDir * 200 )
            .IgnoreGameObject( GameObject )
            .Run();
        
        if ( !trace.Hit || trace.GameObject == null ) return;
        
        // Проверяем, есть ли у объекта интерфейс взаимодействия
        var interactable = trace.GameObject.Components.Get<IInteractable>();
        if ( interactable != null )
        {
            interactable.OnInteract( this );
        }
        
        // Проверяем двери
        var door = trace.GameObject.Components.Get<DarkRPDoor>();
        if ( door != null )
        {
            door.OnPlayerUse( this );
        }
    }
    
    // ============================
    // ЗАРПЛАТА
    // ============================
    
    void HandleSalary()
    {
        float salaryInterval = currentJob?.SalaryInterval ?? 60f;
        
        if ( lastSalary > salaryInterval )
        {
            int salary = currentJob?.Salary ?? 50;
            AddMoney( salary );
            lastSalary = 0;
            
            // Уведомление (локальное)
            Log.Info( $"Зарплата: +${salary}" );
        }
    }
    
    // ============================
    // ПУБЛИЧНЫЕ МЕТОДЫ
    // ============================
    
    /// <summary>Установить профессию</summary>
    public void SetJob( JobDefinition job )
    {
        if ( job == null ) return;
        
        currentJob = job;
        JobName = job.JobName;
        JobColor = job.JobColor;
        
        Log.Info( $"{PlayerName} стал: {job.JobName}" );
    }
    
    /// <summary>Добавить деньги</summary>
    public void AddMoney( int amount )
    {
        Money += amount;
        Log.Info( $"+${amount}. Баланс: ${Money}" );
    }
    
    /// <summary>Потратить деньги (вернёт false если не хватает)</summary>
    public bool SpendMoney( int amount )
    {
        if ( Money < amount )
        {
            Log.Info( $"Недостаточно денег! Нужно: ${amount}, есть: ${Money}" );
            return false;
        }
        
        Money -= amount;
        Log.Info( $"-${amount}. Баланс: ${Money}" );
        return true;
    }
    
    /// <summary>Нанести урон</summary>
    [Broadcast]
    public void TakeDamage( float damage, string attackerName = "" )
    {
        if ( !IsAlive ) return;
        
        // Сначала снимаем броню
        if ( Armor > 0 )
        {
            int armorDamage = (int)MathF.Min( damage * 0.5f, Armor );
            Armor -= armorDamage;
            damage -= armorDamage;
        }
        
        Health -= damage;
        
        if ( Health <= 0 )
        {
            Health = 0;
            Die( attackerName );
        }
    }
    
    /// <summary>Смерть игрока</summary>
    [Broadcast]
    public void Die( string killerName = "" )
    {
        IsAlive = false;
        
        if ( !string.IsNullOrEmpty( killerName ) )
        {
            Log.Info( $"{PlayerName} убит {killerName}" );
        }
        
        // Выбрасываем деньги
        int droppedMoney = Money / 2; // Теряем половину
        Money -= droppedMoney;
        
        // Респавн через 5 секунд
        // (в реальном проекте — таймер или корутина)
    }
    
    /// <summary>Воскресить игрока</summary>
    [Broadcast]
    public void Respawn( Vector3 position )
    {
        IsAlive = true;
        Health = MaxHealth;
        Armor = 0;
        WorldPosition = position;
    }
    
    // ============================
    // ВИЗУАЛ
    // ============================
    
    void UpdateVisuals()
    {
        // Скрываем модель для локального игрока (FPS)
        if ( BodyRenderer != null )
        {
            BodyRenderer.Enabled = IsProxy;
        }
    }
    
    // ============================
    // СЕТЕВЫЕ СОБЫТИЯ
    // ============================
    
    void INetworkListener.OnConnected( Connection connection )
    {
        PlayerName = connection.DisplayName;
        Money = 500;
        Health = MaxHealth;
        IsAlive = true;
        
        Log.Info( $"Игрок {PlayerName} подключился!" );
    }
    
    void INetworkListener.OnDisconnected( Connection connection )
    {
        Log.Info( $"Игрок {PlayerName} отключился!" );
    }
}

/// <summary>
/// Интерфейс для интерактивных объектов.
/// Любой объект, реализующий этот интерфейс, может быть
/// использован игроком по нажатию E.
/// </summary>
public interface IInteractable
{
    /// <summary>Текст подсказки (например, "Нажмите E чтобы открыть")</summary>
    string InteractText { get; }
    
    /// <summary>Вызывается когда игрок нажимает E на объект</summary>
    void OnInteract( DarkRPPlayer player );
}
```

---

## Что здесь важно понимать

### [Sync] — сетевые свойства
Каждое свойство с `[Sync]` автоматически синхронизируется по сети. Когда хост меняет `Money` у игрока — все клиенты видят новое значение.

### [Broadcast] — методы для всех
`[Broadcast]` означает, что метод выполнится на **всех клиентах**. Например, `Die()` — все увидят смерть игрока.

### IsProxy — чужой объект
`IsProxy == true` означает, что это **чужой игрок** на вашем экране. Вы НЕ можете управлять им.

### IInteractable — шаблон взаимодействия
Интерфейс `IInteractable` позволяет создавать **любые интерактивные объекты**. Достаточно реализовать этот интерфейс — и игрок сможет нажать E на объект.

---

**Следующий шаг:** [Система профессий и экономики →](03-JobAndMoney.md)
