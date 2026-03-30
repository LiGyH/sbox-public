# 15. Разрушаемые объекты и осколки

## Обзор

s&box предоставляет систему **разрушаемых объектов** (breakables), которая позволяет создавать объекты, разлетающиеся на части при получении урона. Это используется для:

- **Стёкла** — разбиваются на осколки
- **Ящики** — разлетаются на доски
- **Бочки** — взрываются и разлетаются
- **Мебель** — ломается от ударов
- **Любые декоративные объекты** — вазы, бутылки, горшки

---

## Компонент Prop — базовый разрушаемый объект

Компонент `Prop` — это основной способ создания разрушаемых предметов. Он реализует интерфейс `IDamageable` и поддерживает разрушение с генерацией обломков (gibs).

### Базовая настройка

```csharp
/// <summary>
/// Простой разрушаемый предмет.
/// Добавьте компонент Prop на GameObject с ModelRenderer.
/// </summary>
public sealed class BreakableSetup : Component
{
    protected override void OnAwake()
    {
        var prop = Components.Get<Prop>();
        if ( prop == null ) return;

        // Модель определяет внешний вид и обломки
        prop.Model = Model.Load( "models/props/crate.vmdl" );

        // Здоровье — сколько урона выдержит
        prop.Health = 50f;

        // Статический = не двигается от физики (заборы, стены, витрины)
        prop.IsStatic = false;
    }
}
```

### Обработка разрушения

```csharp
public sealed class BreakableEvents : Component
{
    [Property] public Prop TargetProp { get; set; }

    protected override void OnAwake()
    {
        if ( TargetProp == null ) return;

        // Событие при получении урона
        TargetProp.OnPropTakeDamage = ( DamageInfo info ) =>
        {
            Log.Info( $"Предмет получил {info.Damage} урона! Осталось: {TargetProp.Health}" );
        };

        // Событие при разрушении
        TargetProp.OnPropBreak = () =>
        {
            Log.Info( "Предмет разрушен!" );
            // Здесь можно добавить звук, частицы, очки и т.д.
        };
    }
}
```

---

## Как работают обломки (Gibs)

Когда `Prop` разрушается, он:

1. **Удаляет** оригинальный объект
2. **Создаёт** обломки (gibs) — маленькие модели, определённые в файле модели
3. Обломки **разлетаются** от точки удара
4. Каждый обломок — это `Gib` компонент (наследник `Prop`)
5. Обломки **исчезают** через заданное время (fade)

### Структура ModelBreakPiece

Каждый обломок определяется в файле модели:

```csharp
// Структура обломка (задаётся в модели через ModelDoc)
public struct ModelBreakPiece
{
    public string PieceName;           // Имя обломка
    public string Model;               // Модель обломка (.vmdl)
    public Vector3 Offset;             // Смещение от центра
    public float FadeTime;             // Время до исчезновения (сек)
    public float RandomSpawnChance;    // Шанс появления (0-1)
    public bool IsEssential;           // Обязателен ли (всегда появляется)
    public string CollisionTags;       // Теги для столкновений
    public string PlacementBone;       // Кость привязки
    public string PlacementAttachment; // Точка привязки
    public bool IsClientOnly;          // Только на клиенте (оптимизация)
}
```

### Gib — компонент обломка

```csharp
// Gib наследует Prop и добавляет автоматическое исчезновение
public class Gib : Prop
{
    // Время жизни обломка (в секундах)
    // После этого времени обломок начнёт исчезать
    public float FadeTime { get; set; }
}
```

---

## Создание стекла, которое бьётся по частям

Одна из самых интересных задач — создание **стекла**, которое не разрушается целиком, а **бьётся по отдельным кусочкам**. Рассмотрим несколько подходов.

### Подход 1: Стекло из множества частей (панели)

Разделите стекло на **сетку отдельных объектов**. Каждый кусочек — самостоятельный `Prop`.

```csharp
/// <summary>
/// Стекло, состоящее из множества отдельных панелей.
/// Каждая панель разбивается независимо от удара.
/// </summary>
public sealed class ShatterGlass : Component
{
    [Property] public Model GlassPieceModel { get; set; }
    [Property] public int Columns { get; set; } = 4;
    [Property] public int Rows { get; set; } = 4;
    [Property] public float PieceWidth { get; set; } = 25f;
    [Property] public float PieceHeight { get; set; } = 25f;
    [Property] public float PieceHealth { get; set; } = 5f;
    [Property] public SoundEvent BreakSound { get; set; }
    [Property] public ParticleSystem BreakParticles { get; set; }

    private List<GameObject> pieces = new();

    protected override void OnAwake()
    {
        CreateGlassPieces();
    }

    void CreateGlassPieces()
    {
        // Начальная позиция (верхний левый угол)
        float totalWidth = Columns * PieceWidth;
        float totalHeight = Rows * PieceHeight;
        var startPos = WorldPosition
            - WorldRotation.Right * (totalWidth / 2f)
            + WorldRotation.Up * (totalHeight / 2f);

        for ( int row = 0; row < Rows; row++ )
        {
            for ( int col = 0; col < Columns; col++ )
            {
                var pieceObj = CreateGlassPiece( startPos, row, col );
                pieces.Add( pieceObj );
            }
        }
    }

    GameObject CreateGlassPiece( Vector3 startPos, int row, int col )
    {
        // Позиция кусочка
        var pos = startPos
            + WorldRotation.Right * (col * PieceWidth + PieceWidth / 2f)
            - WorldRotation.Up * (row * PieceHeight + PieceHeight / 2f);

        // Создаём объект
        var pieceObj = new GameObject( true, $"GlassPiece_{row}_{col}" );
        pieceObj.WorldPosition = pos;
        pieceObj.WorldRotation = WorldRotation;
        pieceObj.Parent = GameObject;

        // Рендерер
        var renderer = pieceObj.Components.Create<ModelRenderer>();
        if ( GlassPieceModel != null )
        {
            renderer.Model = GlassPieceModel;
        }
        renderer.Tint = new Color( 0.8f, 0.9f, 1.0f, 0.3f ); // Прозрачное стекло

        // Коллайдер
        var collider = pieceObj.Components.Create<BoxCollider>();
        collider.Scale = new Vector3( PieceWidth, 2f, PieceHeight );

        // Физика (статическая, пока не разбита)
        var body = pieceObj.Components.Create<Rigidbody>();
        body.PhysicsBody.BodyType = PhysicsBodyType.Static;

        // Компонент разбиения
        var breakable = pieceObj.Components.Create<GlassPieceComponent>();
        breakable.Health = PieceHealth;
        breakable.BreakSound = BreakSound;
        breakable.BreakParticles = BreakParticles;

        // Теги для трассировки
        pieceObj.Tags.Add( "solid", "breakable", "glass" );

        return pieceObj;
    }
}

/// <summary>
/// Компонент отдельного кусочка стекла.
/// Получает урон и разбивается независимо.
/// </summary>
public sealed class GlassPieceComponent : Component, Component.IDamageable
{
    [Property] public float Health { get; set; } = 5f;
    [Property] public SoundEvent BreakSound { get; set; }
    [Property] public ParticleSystem BreakParticles { get; set; }

    void IDamageable.OnDamage( in DamageInfo damage )
    {
        Health -= damage.Damage;

        if ( Health <= 0 )
        {
            Break( damage );
        }
    }

    void Break( DamageInfo damage )
    {
        // Звук разбития
        if ( BreakSound != null )
        {
            Sound.Play( BreakSound, WorldPosition );
        }

        // Частицы (осколки)
        if ( BreakParticles != null )
        {
            var fx = new GameObject( true, "GlassBreakFX" );
            fx.WorldPosition = WorldPosition;

            var particle = fx.Components.Create<ParticleEffect>();
            particle.ParticleSystem = BreakParticles;
            fx.DestroyAsync( 3f );
        }

        // Включаем физику и даём импульс обломку
        var body = Components.Get<Rigidbody>();
        if ( body != null )
        {
            body.PhysicsBody.BodyType = PhysicsBodyType.Dynamic;
            body.Gravity = true;

            // Импульс от удара
            var force = damage.Force.Normal * 200f + Vector3.Down * 50f;
            body.ApplyForce( force );
        }

        // Удаляем через некоторое время
        GameObject.DestroyAsync( 3f );
    }
}
```

### Подход 2: Разрушение по зонам

Стекло разделено на **зоны** — при попадании в зону разрушается группа кусочков вокруг точки удара.

```csharp
/// <summary>
/// Стекло, которое разрушается по зонам вокруг точки попадания.
/// При выстреле разбиваются кусочки в определённом радиусе.
/// </summary>
public sealed class ZoneShatterGlass : Component, Component.IDamageable
{
    [Property] public int GridSize { get; set; } = 6;
    [Property] public float TotalWidth { get; set; } = 150f;
    [Property] public float TotalHeight { get; set; } = 100f;
    [Property] public float BreakRadius { get; set; } = 40f;
    [Property] public SoundEvent BreakSound { get; set; }

    // Массив "живых" кусочков (true = целый, false = разбитый)
    private bool[,] grid;
    private List<GameObject> pieceObjects = new();

    protected override void OnAwake()
    {
        grid = new bool[GridSize, GridSize];

        // Все кусочки целые
        for ( int x = 0; x < GridSize; x++ )
            for ( int y = 0; y < GridSize; y++ )
                grid[x, y] = true;

        CreateVisualPieces();
    }

    void CreateVisualPieces()
    {
        float pieceW = TotalWidth / GridSize;
        float pieceH = TotalHeight / GridSize;

        for ( int x = 0; x < GridSize; x++ )
        {
            for ( int y = 0; y < GridSize; y++ )
            {
                var offset = new Vector3(
                    (x - GridSize / 2f + 0.5f) * pieceW,
                    0,
                    (y - GridSize / 2f + 0.5f) * pieceH
                );

                var piece = new GameObject( true, $"Piece_{x}_{y}" );
                piece.WorldPosition = WorldPosition + WorldRotation * offset;
                piece.WorldRotation = WorldRotation;
                piece.Parent = GameObject;

                var renderer = piece.Components.Create<ModelRenderer>();
                renderer.Model = Model.Load( "models/dev/box.vmdl" );
                renderer.Tint = new Color( 0.8f, 0.9f, 1.0f, 0.3f );
                piece.WorldScale = new Vector3( pieceW, 1f, pieceH ) * 0.01f;

                piece.Tags.Add( "solid", "glass" );
                pieceObjects.Add( piece );
            }
        }
    }

    void IDamageable.OnDamage( in DamageInfo damage )
    {
        // Находим точку попадания в локальных координатах
        var localHit = WorldTransform.PointToLocal( damage.Position );
        BreakAtPoint( localHit, damage );
    }

    void BreakAtPoint( Vector3 localPoint, DamageInfo damage )
    {
        float pieceW = TotalWidth / GridSize;
        float pieceH = TotalHeight / GridSize;

        int hitX = (int)((localPoint.x / TotalWidth + 0.5f) * GridSize);
        int hitY = (int)((localPoint.z / TotalHeight + 0.5f) * GridSize);

        // Разбиваем кусочки в радиусе
        for ( int x = 0; x < GridSize; x++ )
        {
            for ( int y = 0; y < GridSize; y++ )
            {
                if ( !grid[x, y] ) continue;

                float dist = MathF.Sqrt(
                    MathF.Pow( (x - hitX) * pieceW, 2 ) +
                    MathF.Pow( (y - hitY) * pieceH, 2 )
                );

                if ( dist <= BreakRadius )
                {
                    BreakPiece( x, y, damage );
                }
            }
        }
    }

    [Broadcast]
    void BreakPiece( int x, int y, DamageInfo damage )
    {
        if ( !grid[x, y] ) return;
        grid[x, y] = false;

        int index = x * GridSize + y;
        if ( index >= 0 && index < pieceObjects.Count )
        {
            var piece = pieceObjects[index];

            // Добавляем физику
            var body = piece.Components.Create<Rigidbody>();
            body.Gravity = true;

            // Импульс от удара
            var force = damage.Force.Normal * 150f + Vector3.Random * 50f;
            body.ApplyForce( force );

            // Удаляем через время
            piece.DestroyAsync( 5f );
        }

        // Звук
        if ( BreakSound != null )
        {
            Sound.Play( BreakSound, WorldPosition );
        }
    }
}
```

---

## Статические разрушаемые объекты

Для объектов, которые **не двигаются** (заборы, стены, витрины), используйте свойство `IsStatic`:

```csharp
/// <summary>
/// Статический разрушаемый забор.
/// Не двигается от физики, но разрушается от урона.
/// </summary>
public sealed class BreakableFence : Component
{
    [Property] public Model FenceModel { get; set; }
    [Property] public float Health { get; set; } = 100f;
    [Property] public SoundEvent BreakSound { get; set; }

    protected override void OnAwake()
    {
        var prop = Components.GetOrCreate<Prop>();
        prop.Model = FenceModel;
        prop.Health = Health;
        prop.IsStatic = true; // Не двигается

        prop.OnPropBreak = () =>
        {
            if ( BreakSound != null )
            {
                Sound.Play( BreakSound, WorldPosition );
            }
        };
    }
}
```

---

## Нанесение урона разрушаемым объектам

### Через трассировку (стрельба)

```csharp
public sealed class DamageDealer : Component
{
    public void ShootAt( Vector3 from, Vector3 direction, float damage )
    {
        var tr = Scene.Trace
            .Ray( from, from + direction * 5000f )
            .WithAllTags( "solid" )
            .Run();

        if ( tr.Hit && tr.GameObject != null )
        {
            // Ищем IDamageable на попавшем объекте
            var damageable = tr.GameObject.Components.GetInParent<IDamageable>();
            if ( damageable != null )
            {
                var info = new DamageInfo( damage, GameObject, GameObject );
                damageable.OnDamage( info );
            }
        }
    }
}
```

### Через взрывной радиус

```csharp
public sealed class ExplosionDamage : Component
{
    public void Explode( Vector3 center, float radius, float damage )
    {
        // Находим все Prop в радиусе
        foreach ( var prop in Scene.GetAllComponents<Prop>() )
        {
            float dist = prop.WorldPosition.Distance( center );
            if ( dist > radius ) continue;

            // Урон уменьшается с расстоянием
            float falloff = 1f - (dist / radius);
            float actualDamage = damage * falloff;

            var info = new DamageInfo( actualDamage, GameObject, GameObject );
            prop.OnDamage( info );
        }
    }
}
```

---

## Полный пример: Витрина магазина

```csharp
/// <summary>
/// Витрина магазина — стекло, которое можно разбить по кусочкам.
/// Размещается на карте как единый объект.
/// </summary>
[Title( "Витрина" )]
[Category( "Мир" )]
[Icon( "window" )]
public sealed class ShopWindow : Component
{
    [Property, Group( "Размер" )] public int Columns { get; set; } = 5;
    [Property, Group( "Размер" )] public int Rows { get; set; } = 3;
    [Property, Group( "Размер" )] public float PanelWidth { get; set; } = 30f;
    [Property, Group( "Размер" )] public float PanelHeight { get; set; } = 30f;

    [Property, Group( "Свойства" )] public float PanelHealth { get; set; } = 10f;
    [Property, Group( "Свойства" )] public Color GlassColor { get; set; } = new Color( 0.9f, 0.95f, 1f, 0.3f );

    [Property, Group( "Эффекты" )] public SoundEvent BreakSound { get; set; }
    [Property, Group( "Эффекты" )] public ParticleSystem ShardParticles { get; set; }

    private GlassPanel[,] panels;

    protected override void OnAwake()
    {
        panels = new GlassPanel[Columns, Rows];
        BuildWindow();
    }

    void BuildWindow()
    {
        float totalW = Columns * PanelWidth;
        float totalH = Rows * PanelHeight;

        for ( int col = 0; col < Columns; col++ )
        {
            for ( int row = 0; row < Rows; row++ )
            {
                var localOffset = new Vector3(
                    (col - Columns / 2f + 0.5f) * PanelWidth,
                    0,
                    (row - Rows / 2f + 0.5f) * PanelHeight
                );

                var worldPos = WorldPosition + WorldRotation * localOffset;

                var panelObj = new GameObject( true, $"Panel_{col}_{row}" );
                panelObj.WorldPosition = worldPos;
                panelObj.WorldRotation = WorldRotation;
                panelObj.Parent = GameObject;
                panelObj.Tags.Add( "solid", "breakable", "glass" );

                // Рендерер
                var renderer = panelObj.Components.Create<ModelRenderer>();
                renderer.Model = Model.Load( "models/dev/box.vmdl" );
                renderer.Tint = GlassColor;
                panelObj.WorldScale = new Vector3( PanelWidth * 0.01f, 0.01f, PanelHeight * 0.01f );

                // Коллайдер
                var collider = panelObj.Components.Create<BoxCollider>();
                collider.Scale = new Vector3( PanelWidth, 2f, PanelHeight );

                // Компонент панели
                var panel = panelObj.Components.Create<GlassPanel>();
                panel.Health = PanelHealth;
                panel.BreakSound = BreakSound;
                panel.ShardParticles = ShardParticles;

                panels[col, row] = panel;
            }
        }
    }

    /// <summary>
    /// Разбить всю витрину (например, от взрыва).
    /// </summary>
    public void BreakAll()
    {
        for ( int col = 0; col < Columns; col++ )
        {
            for ( int row = 0; row < Rows; row++ )
            {
                panels[col, row]?.Break( Vector3.Forward * 100f );
            }
        }
    }
}

/// <summary>
/// Отдельная стеклянная панель. Получает урон и разбивается.
/// </summary>
public sealed class GlassPanel : Component, Component.IDamageable
{
    public float Health { get; set; } = 10f;
    public SoundEvent BreakSound { get; set; }
    public ParticleSystem ShardParticles { get; set; }

    private bool isBroken;

    void IDamageable.OnDamage( in DamageInfo damage )
    {
        if ( isBroken ) return;

        Health -= damage.Damage;

        if ( Health <= 0 )
        {
            Break( damage.Force );
        }
    }

    public void Break( Vector3 force )
    {
        if ( isBroken ) return;
        isBroken = true;

        BroadcastBreak( force );
    }

    [Broadcast]
    void BroadcastBreak( Vector3 force )
    {
        // Звук
        if ( BreakSound != null )
        {
            Sound.Play( BreakSound, WorldPosition );
        }

        // Частицы осколков
        if ( ShardParticles != null )
        {
            var fx = new GameObject( true, "ShardFX" );
            fx.WorldPosition = WorldPosition;

            var particle = fx.Components.Create<ParticleEffect>();
            particle.ParticleSystem = ShardParticles;
            fx.DestroyAsync( 3f );
        }

        // Удаляем панель
        GameObject.Destroy();
    }
}
```

---

## Таблица подходов к разрушению

| Подход | Описание | Когда использовать |
|--------|----------|-------------------|
| **Prop с gibs** | Модель определяет обломки | Ящики, бочки, мебель — разрушаются целиком |
| **Сетка панелей** | Объект из множества мелких частей | Стёкла, окна — бьются по кусочкам |
| **Зональное разрушение** | Разрушение в радиусе от попадания | Большие поверхности, стены |
| **Статический Prop** | Неподвижный, но разрушаемый | Заборы, перила, перегородки |

---

## Следующий шаг

Вернитесь к **[Содержанию](../README.md)** для обзора всех разделов документации.
