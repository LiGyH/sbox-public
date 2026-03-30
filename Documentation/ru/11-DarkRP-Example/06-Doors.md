# DarkRP: Шаг 6 — Система дверей

## DarkRPDoor.cs — Покупаемые двери

В DarkRP двери можно **покупать**, **запирать** и **продавать**. Это основа территориального контроля.

```csharp
// code/Systems/DoorSystem.cs

using Sandbox;

/// <summary>
/// Компонент двери DarkRP.
/// Позволяет покупать, блокировать и разблокировать двери.
/// </summary>
[Title( "DarkRP Дверь" )]
[Category( "DarkRP" )]
[Icon( "door_front" )]
public sealed class DarkRPDoor : Component
{
    // === НАСТРОЙКИ (из редактора) ===
    
    [Property] public string DoorName { get; set; } = "Дверь";
    [Property] public int BuyPrice { get; set; } = 100;
    
    // === СЕТЕВЫЕ СВОЙСТВА ===
    
    /// <summary>ID владельца двери (Guid.Empty = ничья)</summary>
    [Sync] public Guid OwnerId { get; set; }
    
    /// <summary>Имя владельца (для отображения)</summary>
    [Sync] public string OwnerName { get; set; } = "";
    
    /// <summary>Заперта ли дверь</summary>
    [Sync] public bool IsLocked { get; set; } = false;
    
    /// <summary>Открыта ли дверь (анимация)</summary>
    [Sync] public bool IsOpen { get; set; } = false;
    
    // === ПРИВАТНЫЕ ===
    
    private Rotation closedRotation;
    private Rotation openRotation;
    
    protected override void OnAwake()
    {
        closedRotation = WorldRotation;
        openRotation = WorldRotation * Rotation.FromYaw( 90 );
    }
    
    protected override void OnUpdate()
    {
        // Плавная анимация открытия/закрытия
        Rotation target = IsOpen ? openRotation : closedRotation;
        WorldRotation = Rotation.Slerp( WorldRotation, target, Time.Delta * 5f );
    }
    
    /// <summary>Вызывается когда игрок нажимает E на дверь</summary>
    public void OnPlayerUse( DarkRPPlayer player )
    {
        if ( player == null ) return;
        
        // Если дверь ничья — предложить купить
        if ( OwnerId == Guid.Empty )
        {
            TryBuy( player );
            return;
        }
        
        // Если мы владелец — открыть/закрыть
        if ( IsOwner( player ) )
        {
            ToggleDoor();
            return;
        }
        
        // Если заперта — сообщить
        if ( IsLocked )
        {
            Log.Info( $"Дверь заперта! Владелец: {OwnerName}" );
            return;
        }
        
        // Если не заперта — можно открыть
        ToggleDoor();
    }
    
    /// <summary>Попытка купить дверь</summary>
    [Broadcast]
    public void TryBuy( DarkRPPlayer player )
    {
        if ( OwnerId != Guid.Empty )
        {
            Log.Info( "Дверь уже куплена!" );
            return;
        }
        
        if ( !player.SpendMoney( BuyPrice ) )
        {
            Log.Info( $"Недостаточно денег! Нужно: ${BuyPrice}" );
            return;
        }
        
        OwnerId = player.Network.Owner;
        OwnerName = player.PlayerName;
        
        Log.Info( $"{player.PlayerName} купил дверь '{DoorName}' за ${BuyPrice}" );
    }
    
    /// <summary>Продать дверь</summary>
    [Broadcast]
    public void Sell( DarkRPPlayer player )
    {
        if ( !IsOwner( player ) ) return;
        
        int refund = BuyPrice / 2; // Возвращаем половину
        player.AddMoney( refund );
        
        OwnerId = Guid.Empty;
        OwnerName = "";
        IsLocked = false;
        
        Log.Info( $"{player.PlayerName} продал дверь '{DoorName}'. Возврат: ${refund}" );
    }
    
    /// <summary>Запереть/отпереть</summary>
    [Broadcast]
    public void ToggleLock( DarkRPPlayer player )
    {
        if ( !IsOwner( player ) ) return;
        
        IsLocked = !IsLocked;
        Log.Info( IsLocked ? "Дверь заперта!" : "Дверь отперта!" );
    }
    
    /// <summary>Открыть/закрыть дверь</summary>
    [Broadcast]
    public void ToggleDoor()
    {
        if ( IsLocked )
        {
            Log.Info( "Дверь заперта!" );
            return;
        }
        
        IsOpen = !IsOpen;
    }
    
    /// <summary>Проверить владение</summary>
    bool IsOwner( DarkRPPlayer player )
    {
        return player != null && OwnerId == player.Network.Owner;
    }
}
```

---

**Следующий шаг:** [Оружие →](07-Weapons.md)
