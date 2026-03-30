# 9. Встроенные компоненты — Справочник

## Обзор

s&box содержит множество встроенных компонентов, разделённых по категориям. Здесь описаны основные с примерами использования.

---

## 📦 Рендеринг (отображение)

### ModelRenderer — отображение 3D-модели

```csharp
[Title("Пример ModelRenderer")]
public sealed class ModelExample : Component
{
    protected override void OnAwake()
    {
        var renderer = Components.Get<ModelRenderer>();
        
        // Загрузить модель
        renderer.Model = Model.Load( "models/dev/box.vmdl" );
        
        // Цвет/оттенок
        renderer.Tint = Color.Red;
        
        // Включить/выключить тени
        renderer.RenderOptions.CastShadows = true;
    }
}
```

### SkinnedModelRenderer — анимированная модель

```csharp
public sealed class AnimatedModelExample : Component
{
    protected override void OnAwake()
    {
        var skinned = Components.Get<SkinnedModelRenderer>();
        
        // Загрузить модель персонажа
        skinned.Model = Model.Load( "models/citizen/citizen.vmdl" );
        
        // Задать материал (одежду)
        skinned.MaterialOverride = Material.Load( "materials/default.vmat" );
    }
}
```

### SpriteRenderer — 2D-спрайт

```csharp
public sealed class SpriteExample : Component
{
    protected override void OnAwake()
    {
        var sprite = Components.Get<SpriteRenderer>();
        
        // Текстура спрайта
        sprite.Texture = Texture.Load( "textures/particle.vtex" );
        sprite.Size = 32f;
        sprite.Color = Color.White;
    }
}
```

### TextRenderer — 3D текст в мире

```csharp
public sealed class TextExample : Component
{
    protected override void OnAwake()
    {
        var text = Components.Get<TextRenderer>();
        
        text.Text = "Привет, мир!";
        text.Color = Color.White;
        text.FontSize = 32;
    }
}
```

### LineRenderer — линия в пространстве

```csharp
public sealed class LineExample : Component
{
    protected override void OnUpdate()
    {
        var line = Components.Get<LineRenderer>();
        
        // Рисуем линию из нескольких точек
        line.Points = new List<Vector3>
        {
            WorldPosition,
            WorldPosition + Vector3.Forward * 100,
            WorldPosition + Vector3.Forward * 100 + Vector3.Up * 50,
            WorldPosition + Vector3.Forward * 200
        };
        
        line.Color = Color.Red;
        line.Width = 2f;
    }
}
```

### Decal — проецируемая текстура (след пули, кровь)

```csharp
public sealed class DecalExample : Component
{
    public void PlaceBulletHole( Vector3 position, Vector3 normal )
    {
        var decalObj = new GameObject( true, "BulletHole" );
        decalObj.WorldPosition = position;
        decalObj.WorldRotation = Rotation.LookAt( normal );
        
        var decal = decalObj.Components.Create<DecalRenderer>();
        decal.Material = Material.Load( "materials/decals/bullethole.vmat" );
        decal.Size = new Vector3( 4, 4, 4 );
        
        // Удалить через 10 секунд
        decalObj.DestroyAsync( 10f );
    }
}
```

---

## 💡 Освещение

### PointLight — точечный свет

```csharp
public sealed class PointLightExample : Component
{
    protected override void OnAwake()
    {
        var light = Components.Get<PointLight>();
        
        light.LightColor = Color.Orange;
        light.Radius = 500f;        // Дальность света
        light.Brightness = 2f;      // Яркость
        light.Shadows = true;       // Тени
    }
}
```

### SpotLight — прожектор (фонарик)

```csharp
public sealed class FlashlightExample : Component
{
    [Property] public bool IsOn { get; set; } = true;
    
    protected override void OnUpdate()
    {
        var light = Components.Get<SpotLight>();
        if ( light == null ) return;
        
        // Включаем/выключаем по нажатию F
        if ( Input.Pressed( "Flashlight" ) )
        {
            IsOn = !IsOn;
        }
        
        light.Enabled = IsOn;
        
        if ( IsOn )
        {
            light.LightColor = Color.White;
            light.Radius = 1000f;     // Дальность
            light.ConeInner = 15f;    // Внутренний конус (яркий)
            light.ConeOuter = 35f;    // Внешний конус (рассеянный)
            light.Brightness = 3f;
        }
    }
}
```

### DirectionalLight — направленный свет (солнце)

```csharp
public sealed class SunlightExample : Component
{
    [Property] public float TimeOfDay { get; set; } = 12f; // Часы (0-24)
    
    protected override void OnUpdate()
    {
        var sun = Components.Get<DirectionalLight>();
        if ( sun == null ) return;
        
        // Поворачиваем солнце на основе времени суток
        float angle = (TimeOfDay / 24f) * 360f - 90f;
        WorldRotation = Rotation.FromPitch( angle );
        
        // Цвет зависит от времени
        if ( TimeOfDay > 6 && TimeOfDay < 18 )
        {
            // День — белый свет
            sun.LightColor = Color.White;
            sun.Brightness = 3f;
        }
        else
        {
            // Ночь — тёмно-синий
            sun.LightColor = new Color( 0.2f, 0.2f, 0.5f );
            sun.Brightness = 0.5f;
        }
    }
}
```

---

## 🎥 Камера

### CameraComponent — камера

```csharp
public sealed class CameraSetup : Component
{
    protected override void OnAwake()
    {
        var cam = Components.Get<CameraComponent>();
        
        // Основные настройки
        cam.FieldOfView = 90f;        // Угол обзора
        cam.ZNear = 10f;              // Ближняя плоскость отсечения
        cam.ZFar = 10000f;            // Дальняя плоскость
        cam.IsMainCamera = true;      // Это главная камера
        cam.BackgroundColor = Color.Black;
    }
    
    // Пример: плавный зум (прицеливание)
    [Property] public float NormalFov { get; set; } = 90f;
    [Property] public float AimFov { get; set; } = 45f;
    
    protected override void OnUpdate()
    {
        var cam = Components.Get<CameraComponent>();
        if ( cam == null ) return;
        
        float targetFov = Input.Down( "Attack2" ) ? AimFov : NormalFov;
        cam.FieldOfView = MathX.Lerp( cam.FieldOfView, targetFov, Time.Delta * 10f );
    }
}
```

---

## 🔊 Аудио

### Воспроизведение звуков

```csharp
public sealed class AudioExample : Component
{
    [Property] public SoundEvent ShootSound { get; set; }
    [Property] public SoundEvent StepSound { get; set; }
    [Property] public SoundEvent AmbientMusic { get; set; }
    
    private SoundHandle musicHandle;
    
    protected override void OnAwake()
    {
        // Фоновая музыка (зациклена)
        if ( AmbientMusic != null )
        {
            musicHandle = Sound.Play( AmbientMusic, WorldPosition );
            // musicHandle.Volume = 0.5f;
        }
    }
    
    public void PlayShoot()
    {
        if ( ShootSound != null )
        {
            // Звук в точке в пространстве (3D звук)
            Sound.Play( ShootSound, WorldPosition );
        }
    }
    
    public void PlayStep()
    {
        if ( StepSound != null )
        {
            Sound.Play( StepSound, WorldPosition );
        }
    }
    
    protected override void OnDestroy()
    {
        // Останавливаем музыку при уничтожении
        musicHandle.Stop();
    }
}
```

---

## 🗺 Навигация (NavMesh)

### NavMeshAgent — AI перемещение

```csharp
public sealed class AIMovement : Component
{
    [Property] public GameObject Target { get; set; }
    [Property] public float MoveSpeed { get; set; } = 150f;
    [Property] public float StopDistance { get; set; } = 50f;
    
    private NavMeshAgent agent;
    
    protected override void OnAwake()
    {
        agent = Components.Get<NavMeshAgent>();
    }
    
    protected override void OnUpdate()
    {
        if ( agent == null || Target == null ) return;
        
        float distance = WorldPosition.Distance( Target.WorldPosition );
        
        if ( distance > StopDistance )
        {
            // Двигаемся к цели
            agent.MoveTo( Target.WorldPosition );
        }
        else
        {
            // Остановиться рядом с целью
            agent.Stop();
        }
    }
}
```

---

## 🎮 Игровые компоненты

### PlayerController — встроенный контроллер

```csharp
// PlayerController — уже готовый контроллер, предоставляемый s&box
// Обычно вы создаёте свой, но можно использовать встроенный:

public sealed class UsePlayerController : Component
{
    protected override void OnAwake()
    {
        var pc = Components.Get<PlayerController>();
        if ( pc == null ) return;
        
        // Встроенный контроллер имеет базовые настройки:
        // pc.EyeHeight — высота глаз
        // pc.BodyHeight — высота тела
        // pc.BodyRadius — радиус тела
    }
}
```

### SpawnPoint — точка спавна

```csharp
// SpawnPoint — просто маркер в сцене. Используется для определения
// где должны появляться игроки.

public sealed class SpawnSystem : Component
{
    public Vector3 GetRandomSpawnPosition()
    {
        var spawnPoints = Scene.GetAllComponents<SpawnPoint>().ToList();
        
        if ( spawnPoints.Count == 0 )
            return Vector3.Zero;
        
        var random = spawnPoints[Random.Shared.Next( spawnPoints.Count )];
        return random.WorldPosition;
    }
    
    public Rotation GetSpawnRotation( SpawnPoint point )
    {
        return point.WorldRotation;
    }
}
```

---

## 🧱 Карта и мир

### MapInstance — загрузка карты

```csharp
public sealed class MapLoader : Component
{
    [Property] public MapAsset Map { get; set; }
    
    protected override void OnAwake()
    {
        var mapInstance = Components.Get<MapInstance>();
        
        // MapInstance загружает .vmap файл как часть сцены
        // Карта включает геометрию, освещение, коллизии
    }
}
```

### Terrain — ландшафт

```csharp
public sealed class TerrainExample : Component
{
    protected override void OnAwake()
    {
        var terrain = Components.Get<Terrain>();
        
        // Terrain — это ландшафт с высотами
        // Настраивается через редактор (карта высот, текстуры)
    }
}
```

---

## 🎭 Эффекты

### ParticleEffect — частицы

```csharp
public sealed class ParticleExample : Component
{
    [Property] public ParticleSystem ExplosionParticle { get; set; }
    
    public void SpawnExplosion( Vector3 position )
    {
        if ( ExplosionParticle == null ) return;
        
        var obj = new GameObject( true, "Explosion" );
        obj.WorldPosition = position;
        
        var particle = obj.Components.Create<ParticleEffect>();
        particle.ParticleSystem = ExplosionParticle;
        
        // Автоудаление через 5 секунд
        obj.DestroyAsync( 5f );
    }
}
```

---

## 🖥 UI компоненты

### ScreenPanel — 2D UI на экране

```csharp
// Добавьте ScreenPanel на GameObject, затем добавьте
// ваш PanelComponent (Razor-компонент) на тот же объект.
// ScreenPanel создаёт "холст" для 2D UI.

// Типичная настройка:
// GameObject "HUD"
//   ├── ScreenPanel (компонент)
//   └── GameHud (ваш PanelComponent)
```

### WorldPanel — UI в 3D мире

```csharp
// WorldPanel рисует UI в 3D пространстве.
// Полезно для: имён над головой, интерактивных экранов в мире.

// Типичная настройка:
// GameObject "PlayerName" (дочерний у игрока)
//   ├── WorldPanel (компонент)
//   └── NameTag (ваш PanelComponent)
//   LocalPosition = (0, 0, 80) — над головой
```

---

## Таблица всех категорий компонентов

| Категория | Компоненты |
|-----------|-----------|
| **Рендеринг** | ModelRenderer, SkinnedModelRenderer, SpriteRenderer, TextRenderer, LineRenderer, TrailRenderer, DecalRenderer |
| **Освещение** | PointLight, SpotLight, DirectionalLight, AmbientLight, EnvmapProbe |
| **Камера** | CameraComponent, CubemapFog |
| **Физика** | Rigidbody, BoxCollider, SphereCollider, CapsuleCollider, ModelCollider, PlaneCollider, HullCollider, CharacterController |
| **Шарниры** | FixedJoint, HingeJoint, BallJoint, SpringJoint, SliderJoint, WheelJoint |
| **Аудио** | AudioListener, SoundComponent |
| **Навигация** | NavMeshAgent, NavMeshLink, NavMeshArea |
| **Частицы** | ParticleEffect, ParticleEmitter |
| **UI** | ScreenPanel, WorldPanel, WorldInput |
| **Карта** | MapInstance, MapCollider, Terrain |
| **Игровые** | PlayerController, SpawnPoint, Prop |
| **Пост-обработка** | PostProcess, PostProcessVolume |
| **VR** | VRHand, VRTrackedObject, VRModelRenderer |

---

## Следующий шаг

Перейдите к разделу **[Ресурсы и ассеты](../10-Resources-Assets/README.md)** для изучения работы с файлами.
