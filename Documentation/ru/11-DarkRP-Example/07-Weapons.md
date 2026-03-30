# DarkRP: Шаг 7 — Оружие

## BaseWeapon.cs — Базовое оружие

```csharp
// code/Weapons/BaseWeapon.cs

using Sandbox;

/// <summary>
/// Базовый класс оружия для DarkRP.
/// Обрабатывает стрельбу, перезарядку и визуальные эффекты.
/// </summary>
[Title( "Базовое оружие" )]
[Category( "DarkRP" )]
[Icon( "sports_esports" )]
public sealed class BaseWeapon : Component
{
    // === НАСТРОЙКИ ===
    
    [Property, Group( "Основное" )] public string WeaponName { get; set; } = "Пистолет";
    [Property, Group( "Основное" )] public float Damage { get; set; } = 25f;
    [Property, Group( "Основное" )] public float Range { get; set; } = 5000f;
    [Property, Group( "Основное" )] public float FireRate { get; set; } = 0.15f;
    [Property, Group( "Основное" )] public bool IsAutomatic { get; set; } = false;
    
    [Property, Group( "Магазин" )] public int MagazineSize { get; set; } = 12;
    [Property, Group( "Магазин" )] public int MaxReserve { get; set; } = 60;
    [Property, Group( "Магазин" )] public float ReloadTime { get; set; } = 2.0f;
    
    [Property, Group( "Точность" )] public float BaseSpread { get; set; } = 0.5f;
    [Property, Group( "Точность" )] public float MovingSpread { get; set; } = 2.0f;
    [Property, Group( "Точность" )] public float AimSpread { get; set; } = 0.1f;
    
    [Property, Group( "Звуки" )] public SoundEvent ShootSound { get; set; }
    [Property, Group( "Звуки" )] public SoundEvent ReloadSound { get; set; }
    [Property, Group( "Звуки" )] public SoundEvent EmptySound { get; set; }
    
    // === СЕТЕВЫЕ СВОЙСТВА ===
    
    [Sync] public int CurrentAmmo { get; set; }
    [Sync] public int ReserveAmmo { get; set; }
    [Sync] public bool IsReloading { get; set; }
    [Sync] public bool IsAiming { get; set; }
    
    // === ПРИВАТНЫЕ ===
    
    private TimeUntil nextShot;
    private TimeUntil reloadFinish;
    private DarkRPPlayer ownerPlayer;
    
    // ============================
    // ЖИЗНЕННЫЙ ЦИКЛ
    // ============================
    
    protected override void OnAwake()
    {
        CurrentAmmo = MagazineSize;
        ReserveAmmo = MaxReserve;
        ownerPlayer = Components.GetInParent<DarkRPPlayer>();
    }
    
    protected override void OnUpdate()
    {
        if ( IsProxy ) return;
        if ( ownerPlayer == null || !ownerPlayer.IsAlive ) return;
        
        // Прицеливание
        IsAiming = Input.Down( "Attack2" );
        
        // Стрельба
        if ( IsAutomatic ? Input.Down( "Attack1" ) : Input.Pressed( "Attack1" ) )
        {
            TryShoot();
        }
        
        // Перезарядка
        if ( Input.Pressed( "Reload" ) )
        {
            TryReload();
        }
        
        // Завершение перезарядки
        if ( IsReloading && reloadFinish <= 0 )
        {
            FinishReload();
        }
    }
    
    // ============================
    // СТРЕЛЬБА
    // ============================
    
    void TryShoot()
    {
        if ( IsReloading ) return;
        if ( nextShot > 0 ) return;
        
        if ( CurrentAmmo <= 0 )
        {
            // Звук пустого магазина
            PlayEmptySound();
            TryReload();
            return;
        }
        
        Shoot();
        nextShot = FireRate;
    }
    
    void Shoot()
    {
        CurrentAmmo--;
        
        // Рассчитываем разброс
        float spread = GetCurrentSpread();
        
        // Направление стрельбы
        var eyePos = ownerPlayer.CameraObject?.WorldPosition ?? ownerPlayer.WorldPosition;
        var eyeRot = ownerPlayer.EyeAngles.ToRotation();
        var direction = eyeRot.Forward;
        
        // Добавляем разброс
        direction += Vector3.Random * spread * 0.01f;
        direction = direction.Normal;
        
        // Трассировка
        var tr = Scene.Trace
            .Ray( eyePos, eyePos + direction * Range )
            .IgnoreGameObject( ownerPlayer.GameObject )
            .WithoutTags( "trigger" )
            .Run();
        
        if ( tr.Hit && tr.GameObject != null )
        {
            // Наносим урон
            var targetPlayer = tr.GameObject.Components.GetInParent<DarkRPPlayer>();
            if ( targetPlayer != null )
            {
                targetPlayer.TakeDamage( Damage, ownerPlayer.PlayerName );
            }
            
            var health = tr.GameObject.Components.Get<HealthComponent>();
            if ( health != null )
            {
                health.TakeDamage( Damage );
            }
        }
        
        // Эффекты (на всех клиентах)
        BroadcastShoot( eyePos, tr.HitPosition ?? (eyePos + direction * Range) );
    }
    
    float GetCurrentSpread()
    {
        if ( IsAiming ) return AimSpread;
        
        // Проверяем движение
        var cc = ownerPlayer.Components.Get<CharacterController>();
        bool isMoving = cc != null && cc.Velocity.Length > 10f;
        
        return isMoving ? MovingSpread : BaseSpread;
    }
    
    // ============================
    // ПЕРЕЗАРЯДКА
    // ============================
    
    void TryReload()
    {
        if ( IsReloading ) return;
        if ( CurrentAmmo >= MagazineSize ) return;
        if ( ReserveAmmo <= 0 ) return;
        
        StartReload();
    }
    
    [Broadcast]
    void StartReload()
    {
        IsReloading = true;
        reloadFinish = ReloadTime;
        
        // Звук перезарядки
        if ( ReloadSound != null )
        {
            Sound.Play( ReloadSound, WorldPosition );
        }
        
        Log.Info( "Перезарядка..." );
    }
    
    void FinishReload()
    {
        IsReloading = false;
        
        int needed = MagazineSize - CurrentAmmo;
        int available = System.Math.Min( needed, ReserveAmmo );
        
        CurrentAmmo += available;
        ReserveAmmo -= available;
        
        Log.Info( $"Перезарядка завершена! {CurrentAmmo}/{MagazineSize} (запас: {ReserveAmmo})" );
    }
    
    // ============================
    // ЭФФЕКТЫ
    // ============================
    
    [Broadcast]
    void BroadcastShoot( Vector3 from, Vector3 to )
    {
        // Звук выстрела
        if ( ShootSound != null )
        {
            Sound.Play( ShootSound, WorldPosition );
        }
    }
    
    [Broadcast]
    void PlayEmptySound()
    {
        if ( EmptySound != null )
        {
            Sound.Play( EmptySound, WorldPosition );
        }
    }
}
```

---

## Важно: WorldModel, ViewModel и Prediction

Пример выше показывает **базовое** оружие. В реальной игре оружие включает:

- **WorldModel** — модель оружия, которую видят **другие игроки** (в мире, от третьего лица)
- **ViewModel** — модель с руками, которую видит **только владелец** (от первого лица, на отдельном слое рендеринга)
- **Prediction (предсказание)** — мгновенное проигрывание эффектов на клиенте без ожидания сервера

Подробнее об этих темах — в разделе **[Оружие: WorldModel, ViewModel и Prediction](../13-Weapons-Advanced/README.md)**.

---

**Следующий шаг:** [Главный менеджер игры →](08-GameManager.md)
