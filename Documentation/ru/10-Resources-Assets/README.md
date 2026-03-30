# 10. Ресурсы и ассеты

## Что такое ресурсы?

**Ресурсы (Assets)** — это файлы, которые ваша игра использует:
- **Модели** (.vmdl) — 3D модели
- **Материалы** (.vmat) — текстуры и свойства поверхностей
- **Звуки** (.sound) — звуковые эффекты и музыка
- **Текстуры** (.vtex) — изображения
- **Частицы** (.vpcf) — системы частиц
- **Сцены** (.scene) — сохранённые сцены
- **Префабы** (.prefab) — заготовки объектов

---

## Загрузка ресурсов из кода

### Модели

```csharp
public sealed class ModelLoadExample : Component
{
    // Способ 1: Через свойство (перетащите в редакторе)
    [Property] public Model MyModel { get; set; }
    
    // Способ 2: Загрузка из кода
    protected override void OnAwake()
    {
        var renderer = Components.Get<ModelRenderer>();
        
        // Загрузка модели по пути
        renderer.Model = Model.Load( "models/dev/box.vmdl" );
        
        // Встроенные модели для разработки:
        // "models/dev/box.vmdl"       — куб
        // "models/dev/sphere.vmdl"    — сфера
        // "models/dev/plane.vmdl"     — плоскость
        // "models/citizen/citizen.vmdl" — модель человека
    }
}
```

### Материалы

```csharp
public sealed class MaterialExample : Component
{
    [Property] public Material MyMaterial { get; set; }
    
    protected override void OnAwake()
    {
        var renderer = Components.Get<ModelRenderer>();
        
        // Загрузка материала
        renderer.MaterialOverride = Material.Load( "materials/default.vmat" );
        
        // Изменение цвета
        renderer.Tint = Color.Red;
    }
}
```

### Звуки

```csharp
public sealed class SoundLoadExample : Component
{
    // Способ 1: Ссылка на звуковое событие
    [Property] public SoundEvent ShootSound { get; set; }
    
    public void PlaySound()
    {
        if ( ShootSound != null )
        {
            // Воспроизвести в позиции
            Sound.Play( ShootSound, WorldPosition );
        }
    }
}
```

### Текстуры

```csharp
public sealed class TextureExample : Component
{
    [Property] public Texture MyTexture { get; set; }
    
    protected override void OnAwake()
    {
        // Загрузить текстуру
        var tex = Texture.Load( "textures/mytexture.vtex" );
    }
}
```

---

## Префабы (Prefabs)

Префабы — это **заготовки** объектов. Вы настраиваете объект в редакторе, сохраняете как префаб, и затем создаёте копии из кода.

### Создание из префаба

```csharp
public sealed class PrefabExample : Component
{
    // Ссылка на префаб (перетащите в редакторе)
    [Property] public PrefabFile EnemyPrefab { get; set; }
    [Property] public PrefabFile BulletPrefab { get; set; }
    [Property] public PrefabFile ExplosionPrefab { get; set; }
    
    // Создать врага
    public GameObject SpawnEnemy( Vector3 position )
    {
        if ( EnemyPrefab == null ) return null;
        
        var enemy = SceneUtility.Instantiate( EnemyPrefab, position, Rotation.Identity );
        return enemy;
    }
    
    // Создать пулю с начальной скоростью
    public void SpawnBullet( Vector3 position, Vector3 direction, float speed )
    {
        if ( BulletPrefab == null ) return;
        
        var bullet = SceneUtility.Instantiate( 
            BulletPrefab, 
            position, 
            Rotation.LookAt( direction ) 
        );
        
        // Настраиваем после создания
        var bulletComp = bullet.Components.Get<BulletComponent>();
        if ( bulletComp != null )
        {
            bulletComp.Speed = speed;
            bulletComp.Direction = direction;
        }
    }
    
    // Создать взрыв (эффект)
    public void SpawnExplosion( Vector3 position )
    {
        if ( ExplosionPrefab == null ) return;
        
        var fx = SceneUtility.Instantiate( ExplosionPrefab, position, Rotation.Identity );
        
        // Автоудаление через 3 секунды
        fx.DestroyAsync( 3f );
    }
}
```

---

## GameResource — пользовательские ресурсы

Вы можете создавать **свои типы ресурсов**, которые редактируются в редакторе:

### Определение ресурса

```csharp
/// <summary>
/// Определение оружия. Создаётся как файл .weapon в редакторе.
/// </summary>
[GameResource( "Weapon Definition", "weapon", "Определение оружия" )]
public class WeaponDefinition : GameResource
{
    [Property] public string WeaponName { get; set; } = "Пистолет";
    [Property] public Model WorldModel { get; set; }
    [Property] public SoundEvent ShootSound { get; set; }
    [Property] public SoundEvent ReloadSound { get; set; }
    [Property, Range( 1, 200 )] public float Damage { get; set; } = 25f;
    [Property, Range( 0.01f, 2f )] public float FireRate { get; set; } = 0.1f;
    [Property] public int MagazineSize { get; set; } = 30;
    [Property] public int MaxAmmo { get; set; } = 120;
    [Property, Range( 100, 10000 )] public float Range { get; set; } = 5000f;
    [Property, Range( 0, 10 )] public float Spread { get; set; } = 1f;
}
```

### Использование ресурса

```csharp
public sealed class Weapon : Component
{
    // Перетащите файл .weapon в это поле
    [Property] public WeaponDefinition Definition { get; set; }
    
    private int currentAmmo;
    private int reserveAmmo;
    private TimeUntil nextShot;
    
    protected override void OnAwake()
    {
        if ( Definition == null ) return;
        
        currentAmmo = Definition.MagazineSize;
        reserveAmmo = Definition.MaxAmmo;
        
        // Загружаем модель оружия
        var renderer = Components.Get<ModelRenderer>();
        if ( renderer != null )
        {
            renderer.Model = Definition.WorldModel;
        }
    }
    
    protected override void OnUpdate()
    {
        if ( Definition == null ) return;
        if ( IsProxy ) return;
        
        if ( Input.Down( "Attack1" ) && nextShot <= 0 && currentAmmo > 0 )
        {
            Shoot();
            nextShot = Definition.FireRate;
        }
        
        if ( Input.Pressed( "Reload" ) )
        {
            Reload();
        }
    }
    
    void Shoot()
    {
        currentAmmo--;
        
        // Звук
        if ( Definition.ShootSound != null )
        {
            Sound.Play( Definition.ShootSound, WorldPosition );
        }
        
        // Трассировка
        var spread = Vector3.Random * Definition.Spread;
        var direction = WorldRotation.Forward + spread;
        
        var trace = Scene.Trace
            .Ray( WorldPosition, WorldPosition + direction * Definition.Range )
            .IgnoreGameObject( GameObject )
            .Run();
        
        if ( trace.Hit )
        {
            var health = trace.GameObject?.Components.Get<HealthComponent>();
            health?.TakeDamage( Definition.Damage );
        }
    }
    
    void Reload()
    {
        int needed = Definition.MagazineSize - currentAmmo;
        int available = Math.Min( needed, reserveAmmo );
        
        currentAmmo += available;
        reserveAmmo -= available;
        
        if ( Definition.ReloadSound != null )
        {
            Sound.Play( Definition.ReloadSound, WorldPosition );
        }
    }
}
```

### Определение NPC

```csharp
[GameResource( "NPC Definition", "npc", "Определение NPC" )]
public class NPCDefinition : GameResource
{
    [Property] public string NPCName { get; set; } = "Враг";
    [Property] public Model Model { get; set; }
    [Property] public float Health { get; set; } = 100f;
    [Property] public float MoveSpeed { get; set; } = 150f;
    [Property] public float AttackDamage { get; set; } = 10f;
    [Property] public float AttackRange { get; set; } = 100f;
    [Property] public float DetectionRange { get; set; } = 500f;
    [Property] public Color NameColor { get; set; } = Color.Red;
    [Property] public SoundEvent DeathSound { get; set; }
    [Property] public List<string> LootTable { get; set; } = new();
}
```

---

## Файловые пути

### Структура проекта

```
МойПроект/
├── code/                      ← Код (C#, Razor)
│   ├── Components/
│   │   └── MyComponent.cs
│   ├── UI/
│   │   ├── Hud.razor          ← UI шаблон
│   │   └── Hud.razor.scss     ← Стили (ОБЯЗАТЕЛЬНО .razor.scss!)
│   └── Resources/
│       ├── WeaponDefs/
│       │   ├── pistol.weapon  ← Файл GameResource
│       │   └── rifle.weapon
│       └── NPCDefs/
│           └── zombie.npc
├── models/                    ← 3D модели (.vmdl)
├── materials/                 ← Материалы (.vmat)
├── textures/                  ← Текстуры (.vtex)
├── sounds/                    ← Звуки (.sound)
├── particles/                 ← Частицы (.vpcf)
└── scenes/                    ← Сцены (.scene)
```

### Правила путей

```csharp
// Пути к ресурсам — относительные от корня проекта
Model.Load( "models/dev/box.vmdl" );           // Модель
Material.Load( "materials/default.vmat" );       // Материал
Texture.Load( "textures/mytexture.vtex" );       // Текстура

// Для Razor файлов — стили РЯДОМ с .razor файлом
// code/UI/Hud.razor       → code/UI/Hud.razor.scss
// code/UI/Menu.razor      → code/UI/Menu.razor.scss
```

---

## Следующий шаг

Перейдите к разделу **[Пример проекта: DarkRP](../11-DarkRP-Example/README.md)** — полный пошаговый проект!
