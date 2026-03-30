# DarkRP: Шаг 3 — Система профессий и экономики

## JobSystem.cs — Управление профессиями

```csharp
// code/Systems/JobSystem.cs

using Sandbox;
using System.Collections.Generic;
using System.Linq;

/// <summary>
/// Управляет всеми профессиями в игре.
/// Загружает определения, проверяет ограничения, назначает профессии.
/// </summary>
[Title( "Система профессий" )]
[Category( "DarkRP" )]
[Icon( "work" )]
public sealed class JobSystem : Component
{
    /// <summary>Список всех доступных профессий</summary>
    [Property]
    public List<JobDefinition> AvailableJobs { get; set; } = new();
    
    /// <summary>Профессия по умолчанию (для новых игроков)</summary>
    [Property]
    public JobDefinition DefaultJob { get; set; }
    
    /// <summary>Получить все профессии по категориям</summary>
    public Dictionary<string, List<JobDefinition>> GetJobsByCategory()
    {
        var result = new Dictionary<string, List<JobDefinition>>();
        
        foreach ( var job in AvailableJobs )
        {
            if ( !result.ContainsKey( job.Category ) )
            {
                result[job.Category] = new List<JobDefinition>();
            }
            result[job.Category].Add( job );
        }
        
        return result;
    }
    
    /// <summary>Можно ли игроку выбрать эту профессию?</summary>
    public bool CanSelectJob( DarkRPPlayer player, JobDefinition job )
    {
        if ( job == null ) return false;
        
        // Проверяем лимит игроков
        if ( job.MaxPlayers > 0 )
        {
            int currentCount = Scene.GetAllComponents<DarkRPPlayer>()
                .Count( p => p.JobName == job.JobName );
            
            if ( currentCount >= job.MaxPlayers )
            {
                Log.Info( $"Профессия {job.JobName} заполнена ({currentCount}/{job.MaxPlayers})" );
                return false;
            }
        }
        
        return true;
    }
    
    /// <summary>Назначить профессию игроку</summary>
    [Broadcast]
    public void AssignJob( DarkRPPlayer player, string jobName )
    {
        if ( player == null ) return;
        
        var job = AvailableJobs.FirstOrDefault( j => j.JobName == jobName );
        if ( job == null )
        {
            Log.Warning( $"Профессия не найдена: {jobName}" );
            return;
        }
        
        if ( !CanSelectJob( player, job ) )
        {
            return;
        }
        
        player.SetJob( job );
        
        // Уведомление всем
        Log.Info( $"{player.PlayerName} теперь {job.JobName}" );
    }
    
    /// <summary>Получить количество игроков с профессией</summary>
    public int GetJobPlayerCount( string jobName )
    {
        return Scene.GetAllComponents<DarkRPPlayer>()
            .Count( p => p.JobName == jobName );
    }
}
```

---

## MoneySystem.cs — Экономика

```csharp
// code/Systems/MoneySystem.cs

using Sandbox;
using System.Linq;

/// <summary>
/// Управляет экономикой: переводы, выброс денег, штрафы.
/// </summary>
[Title( "Экономика" )]
[Category( "DarkRP" )]
[Icon( "payments" )]
public sealed class MoneySystem : Component
{
    /// <summary>Перевести деньги другому игроку</summary>
    [Broadcast]
    public void TransferMoney( DarkRPPlayer from, DarkRPPlayer to, int amount )
    {
        if ( from == null || to == null ) return;
        if ( amount <= 0 ) return;
        
        if ( from.Money < amount )
        {
            Log.Info( "Недостаточно средств для перевода!" );
            return;
        }
        
        from.Money -= amount;
        to.Money += amount;
        
        Log.Info( $"{from.PlayerName} перевёл ${amount} игроку {to.PlayerName}" );
    }
    
    /// <summary>Выбросить деньги на землю</summary>
    [Broadcast]
    public void DropMoney( DarkRPPlayer player, int amount )
    {
        if ( player == null ) return;
        if ( amount <= 0 || player.Money < amount ) return;
        
        player.Money -= amount;
        
        // Создаём объект денег на земле
        SpawnMoneyDrop( player.WorldPosition + Vector3.Up * 50, amount );
        
        Log.Info( $"{player.PlayerName} выбросил ${amount}" );
    }
    
    /// <summary>Создать объект денег в мире</summary>
    void SpawnMoneyDrop( Vector3 position, int amount )
    {
        var moneyObj = new GameObject( true, $"Money_{amount}" );
        moneyObj.WorldPosition = position;
        
        // Добавляем визуал
        var renderer = moneyObj.Components.Create<ModelRenderer>();
        renderer.Model = Model.Load( "models/dev/box.vmdl" );
        renderer.Tint = Color.Green;
        moneyObj.LocalScale = new Vector3( 0.2f, 0.4f, 0.05f );
        
        // Добавляем физику
        var collider = moneyObj.Components.Create<BoxCollider>();
        collider.Scale = new Vector3( 10, 20, 3 );
        
        var body = moneyObj.Components.Create<Rigidbody>();
        body.Gravity = true;
        
        // Добавляем компонент денег
        var moneyComp = moneyObj.Components.Create<DroppedMoney>();
        moneyComp.Amount = amount;
        
        // Регистрируем в сети
        moneyObj.NetworkSpawn();
    }
    
    /// <summary>Оштрафовать игрока (для полиции)</summary>
    [Broadcast]
    public void FinePlayer( DarkRPPlayer police, DarkRPPlayer target, int amount, string reason )
    {
        if ( police == null || target == null ) return;
        
        // Проверяем, может ли полицейский штрафовать
        // (в реальном проекте — через JobDefinition)
        
        int actualFine = System.Math.Min( amount, target.Money );
        target.Money -= actualFine;
        
        Log.Info( $"{police.PlayerName} оштрафовал {target.PlayerName} на ${actualFine}. Причина: {reason}" );
    }
}
```

---

## DroppedMoney.cs — Деньги на земле

```csharp
// code/Entities/DroppedMoney.cs

using Sandbox;

/// <summary>
/// Деньги, выброшенные на землю.
/// Можно подобрать, нажав E.
/// </summary>
[Title( "Выброшенные деньги" )]
[Category( "DarkRP" )]
[Icon( "attach_money" )]
public sealed class DroppedMoney : Component, IInteractable
{
    [Sync] public int Amount { get; set; } = 100;
    
    // Текст подсказки при наведении
    public string InteractText => $"Подобрать ${Amount}";
    
    // Вызывается когда игрок нажимает E
    public void OnInteract( DarkRPPlayer player )
    {
        if ( player == null ) return;
        
        player.AddMoney( Amount );
        Log.Info( $"{player.PlayerName} подобрал ${Amount}" );
        
        // Удаляем объект
        GameObject.Destroy();
    }
    
    protected override void OnUpdate()
    {
        // Вращаем для красоты
        WorldRotation *= Rotation.FromAxis( Vector3.Up, 90f * Time.Delta );
    }
}
```

---

**Следующий шаг:** [Игровые сущности →](04-Entities.md)
