# DarkRP: Шаг 4 — Игровые сущности

## MoneyPrinter.cs — Принтер денег

Принтер денег — культовый объект DarkRP. Он генерирует деньги со временем.

```csharp
// code/Entities/MoneyPrinter.cs

using Sandbox;

/// <summary>
/// Принтер денег — генерирует деньги каждые несколько секунд.
/// Можно купить в магазине. Может перегреться и загореться!
/// </summary>
[Title( "Принтер денег" )]
[Category( "DarkRP" )]
[Icon( "print" )]
public sealed class MoneyPrinter : Component, IInteractable
{
    // === НАСТРОЙКИ ===
    
    [Property, Sync] public string PrinterName { get; set; } = "Принтер денег";
    [Property] public int MoneyPerCycle { get; set; } = 25;
    [Property] public float CycleTime { get; set; } = 30f; // Секунд
    [Property] public int MaxStorage { get; set; } = 500;
    [Property] public float OverheatChance { get; set; } = 0.05f; // 5% за цикл
    
    // === СЕТЕВЫЕ СВОЙСТВА ===
    
    [Sync] public int StoredMoney { get; set; } = 0;
    [Sync] public bool IsOverheated { get; set; } = false;
    [Sync] public bool IsOnFire { get; set; } = false;
    [Sync] public Guid OwnerId { get; set; }
    
    // === ПРИВАТНЫЕ ===
    
    private TimeSince lastPrint;
    
    // === ИНТЕРФЕЙС ВЗАИМОДЕЙСТВИЯ ===
    
    public string InteractText => StoredMoney > 0 
        ? $"Забрать ${StoredMoney} из принтера" 
        : "Принтер пуст";
    
    public void OnInteract( DarkRPPlayer player )
    {
        if ( StoredMoney <= 0 ) return;
        
        // Забираем деньги из принтера
        player.AddMoney( StoredMoney );
        Log.Info( $"{player.PlayerName} забрал ${StoredMoney} из принтера" );
        StoredMoney = 0;
    }
    
    // === ЛОГИКА ===
    
    protected override void OnUpdate()
    {
        if ( IsProxy ) return;
        if ( IsOnFire ) return;
        
        if ( lastPrint > CycleTime )
        {
            lastPrint = 0;
            PrintMoney();
        }
    }
    
    void PrintMoney()
    {
        // Проверяем перегрев
        if ( Random.Shared.NextSingle() < OverheatChance )
        {
            Overheat();
            return;
        }
        
        // Печатаем деньги
        int toPrint = System.Math.Min( MoneyPerCycle, MaxStorage - StoredMoney );
        StoredMoney += toPrint;
        
        Log.Info( $"Принтер напечатал ${toPrint}. В хранилище: ${StoredMoney}/{MaxStorage}" );
    }
    
    void Overheat()
    {
        IsOverheated = true;
        Log.Warning( "Принтер перегрелся!" );
        
        // Через некоторое время может загореться
        // (в реальном проекте — таймер и анимации)
    }
    
    /// <summary>Потушить принтер</summary>
    [Broadcast]
    public void Extinguish()
    {
        IsOnFire = false;
        IsOverheated = false;
        Log.Info( "Принтер потушен!" );
    }
}
```

---

## ShopNPC.cs — NPC-продавец

```csharp
// code/Entities/ShopNPC.cs

using Sandbox;
using System.Collections.Generic;

/// <summary>
/// NPC-продавец. При нажатии E открывает магазин.
/// </summary>
[Title( "NPC Продавец" )]
[Category( "DarkRP" )]
[Icon( "storefront" )]
public sealed class ShopNPC : Component, IInteractable
{
    [Property] public string ShopName { get; set; } = "Оружейный магазин";
    [Property] public List<ShopItem> Items { get; set; } = new();
    
    public string InteractText => $"Открыть {ShopName}";
    
    public void OnInteract( DarkRPPlayer player )
    {
        Log.Info( $"{player.PlayerName} открыл {ShopName}" );
        // Здесь нужно открыть UI магазина
        // Это делается через систему событий или прямой вызов UI
    }
    
    /// <summary>Купить предмет из магазина</summary>
    [Broadcast]
    public void BuyItem( DarkRPPlayer player, int itemIndex )
    {
        if ( player == null ) return;
        if ( itemIndex < 0 || itemIndex >= Items.Count ) return;
        
        var item = Items[itemIndex];
        
        if ( !player.SpendMoney( item.Price ) )
        {
            Log.Info( "Недостаточно денег!" );
            return;
        }
        
        // Выдаём предмет
        GiveItem( player, item );
        
        Log.Info( $"{player.PlayerName} купил {item.Name} за ${item.Price}" );
    }
    
    void GiveItem( DarkRPPlayer player, ShopItem item )
    {
        switch ( item.Type )
        {
            case ShopItemType.Weapon:
                // Создать оружие и дать игроку
                Log.Info( $"Выдано оружие: {item.Name}" );
                break;
                
            case ShopItemType.Health:
                player.Health = System.MathF.Min( player.Health + 50, player.MaxHealth );
                Log.Info( $"Здоровье восстановлено!" );
                break;
                
            case ShopItemType.Armor:
                player.Armor = System.Math.Min( player.Armor + 50, 100 );
                Log.Info( $"Броня выдана!" );
                break;
                
            case ShopItemType.Entity:
                // Спавним сущность (например, принтер денег)
                SpawnPurchasedEntity( player, item );
                break;
        }
    }
    
    void SpawnPurchasedEntity( DarkRPPlayer player, ShopItem item )
    {
        var pos = player.WorldPosition + player.WorldRotation.Forward * 100;
        
        if ( item.Name == "Принтер денег" )
        {
            var printerObj = new GameObject( true, "MoneyPrinter" );
            printerObj.WorldPosition = pos;
            
            var renderer = printerObj.Components.Create<ModelRenderer>();
            renderer.Model = Model.Load( "models/dev/box.vmdl" );
            renderer.Tint = new Color( 0.2f, 0.8f, 0.2f );
            
            var collider = printerObj.Components.Create<BoxCollider>();
            collider.Scale = new Vector3( 30, 30, 20 );
            
            var printer = printerObj.Components.Create<MoneyPrinter>();
            printer.OwnerId = player.Network.Owner;
            
            printerObj.NetworkSpawn();
        }
    }
}

/// <summary>Предмет в магазине</summary>
public class ShopItem
{
    public string Name { get; set; }
    public string Description { get; set; }
    public int Price { get; set; }
    public ShopItemType Type { get; set; }
    public string ResourcePath { get; set; }
}

/// <summary>Тип предмета</summary>
public enum ShopItemType
{
    Weapon,
    Health,
    Armor,
    Entity,
    Tool
}
```

---

**Следующий шаг:** [Интерфейс (HUD, меню) →](05-UI.md)
