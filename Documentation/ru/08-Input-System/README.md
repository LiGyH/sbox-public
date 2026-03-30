# 8. Система ввода (Input)

## Обзор

Система ввода в s&box обрабатывает **клавиатуру**, **мышь** и **геймпад**. Все действия определяются через **именованные действия** (Actions), такие как "Jump", "Attack1" и т.д.

---

## Именованные действия (Actions)

s&box использует систему **именованных действий** вместо прямой проверки клавиш. Это позволяет игрокам переназначать кнопки.

### Стандартные действия

| Действие | Клавиша | Геймпад | Описание |
|----------|---------|---------|----------|
| `Forward` | W | Left Stick Up | Вперёд |
| `Backward` | S | Left Stick Down | Назад |
| `Left` | A | Left Stick Left | Влево |
| `Right` | D | Left Stick Right | Вправо |
| `Jump` | Space | A | Прыжок |
| `Run` | Shift | Left Stick Click | Бег |
| `Duck` | Ctrl | B | Присесть |
| `Attack1` | ЛКМ | Right Trigger | Основная атака |
| `Attack2` | ПКМ | Left Trigger | Дополнительная атака |
| `Reload` | R | X | Перезарядка |
| `Use` | E | Y | Использовать |
| `Flashlight` | F | D-Pad Up | Фонарик |
| `Drop` | G | — | Выбросить |
| `Voice` | V | — | Голосовой чат |
| `Score` | Tab | Left Menu | Таблица очков |
| `Menu` | Q | Right Menu | Меню |
| `Chat` | Enter | — | Чат |
| `Slot1`-`Slot0` | 1-0 | D-Pad | Слоты инвентаря |
| `SlotPrev` | Mouse4 | Left Bumper | Предыдущий слот |
| `SlotNext` | Mouse5 | Right Bumper | Следующий слот |
| `View` | C | Right Stick Click | Вид |

---

## Проверка нажатий

### Основные методы

```csharp
public sealed class InputExample : Component
{
    protected override void OnUpdate()
    {
        // === НАЖАТИЕ (один раз при нажатии) ===
        if ( Input.Pressed( "Jump" ) )
        {
            Log.Info( "Прыжок!" );
            // Вызывается ОДИН РАЗ в момент нажатия
        }
        
        // === УДЕРЖАНИЕ (каждый кадр пока зажато) ===
        if ( Input.Down( "Attack1" ) )
        {
            Log.Info( "Стреляем (автоматический огонь)!" );
            // Вызывается КАЖДЫЙ КАДР пока кнопка зажата
        }
        
        // === ОТПУСКАНИЕ (один раз при отпускании) ===
        if ( Input.Released( "Attack1" ) )
        {
            Log.Info( "Перестали стрелять!" );
            // Вызывается ОДИН РАЗ когда кнопка отпущена
        }
    }
}
```

### Примеры использования

```csharp
public sealed class InputPracticalExample : Component
{
    [Property] public float FireRate { get; set; } = 0.1f;
    
    private TimeUntil nextShot;
    private int selectedSlot = 0;
    
    protected override void OnUpdate()
    {
        // === СТРЕЛЬБА С ЗАДЕРЖКОЙ ===
        if ( Input.Down( "Attack1" ) && nextShot <= 0 )
        {
            Shoot();
            nextShot = FireRate;
        }
        
        // === ОДИНОЧНОЕ ДЕЙСТВИЕ ===
        if ( Input.Pressed( "Reload" ) )
        {
            Reload();
        }
        
        if ( Input.Pressed( "Use" ) )
        {
            Interact();
        }
        
        // === ПЕРЕКЛЮЧЕНИЕ СЛОТОВ ===
        for ( int i = 1; i <= 9; i++ )
        {
            if ( Input.Pressed( $"Slot{i}" ) )
            {
                selectedSlot = i;
                Log.Info( $"Выбран слот: {selectedSlot}" );
            }
        }
        
        // Прокрутка слотов
        if ( Input.Pressed( "SlotNext" ) )
        {
            selectedSlot = (selectedSlot + 1) % 10;
        }
        if ( Input.Pressed( "SlotPrev" ) )
        {
            selectedSlot = (selectedSlot - 1 + 10) % 10;
        }
        
        // === УСЛОВНЫЕ ДЕЙСТВИЯ ===
        if ( Input.Down( "Run" ) )
        {
            // Бежим
        }
        else if ( Input.Down( "Duck" ) )
        {
            // Приседаем
        }
        else
        {
            // Идём
        }
        
        // === ТАБЛИЦА ОЧКОВ (показываем пока зажат Tab) ===
        bool showScoreboard = Input.Down( "Score" );
    }
    
    void Shoot() { Log.Info( "Бам!" ); }
    void Reload() { Log.Info( "Перезарядка..." ); }
    void Interact() { Log.Info( "Взаимодействие!" ); }
}
```

---

## Аналоговый ввод

### AnalogMove — движение (WASD / стик)

```csharp
public sealed class AnalogMoveExample : Component
{
    [Property] public float Speed { get; set; } = 200f;
    
    protected override void OnUpdate()
    {
        // Input.AnalogMove — вектор направления движения
        // x: вперёд/назад (-1 до 1)
        // y: влево/вправо (-1 до 1)
        // z: вверх/вниз (0 обычно)
        
        Vector3 moveDir = Input.AnalogMove;
        
        // Если ничего не нажато — moveDir == Vector3.Zero
        if ( moveDir.Length > 0 )
        {
            // Преобразуем направление относительно вращения объекта
            Vector3 worldMoveDir = WorldRotation * moveDir;
            
            // Двигаем объект
            WorldPosition += worldMoveDir * Speed * Time.Delta;
        }
        
        // === ОТДЕЛЬНЫЕ ОСИ ===
        float forward = moveDir.x;  // W/S: +1 вперёд, -1 назад
        float strafe = moveDir.y;   // A/D: +1 влево, -1 вправо
        
        Log.Info( $"Вперёд: {forward:F1}, Вбок: {strafe:F1}" );
    }
}
```

### AnalogLook — обзор (мышь / стик)

```csharp
public sealed class AnalogLookExample : Component
{
    [Property] public float Sensitivity { get; set; } = 1.0f;
    [Property] public GameObject CameraObject { get; set; }
    
    private Angles eyeAngles;
    
    protected override void OnUpdate()
    {
        // Input.AnalogLook — движение мыши / правый стик
        // yaw: горизонтальный поворот
        // pitch: вертикальный наклон
        
        var look = Input.AnalogLook;
        
        // Обновляем углы камеры
        eyeAngles.yaw -= look.yaw * Sensitivity;
        eyeAngles.pitch += look.pitch * Sensitivity;
        
        // Ограничиваем наклон (нельзя посмотреть за спину)
        eyeAngles.pitch = eyeAngles.pitch.Clamp( -89f, 89f );
        eyeAngles.roll = 0; // Без крена
        
        // Применяем к камере (полное вращение)
        if ( CameraObject != null )
        {
            CameraObject.WorldRotation = eyeAngles.ToRotation();
        }
        
        // Применяем к телу (только горизонтальный поворот)
        WorldRotation = Rotation.FromYaw( eyeAngles.yaw );
    }
}
```

---

## Мышь

```csharp
public sealed class MouseExample : Component
{
    protected override void OnUpdate()
    {
        // Позиция мыши на экране (в пикселях)
        Vector2 mousePos = Mouse.Position;
        
        // Дельта мыши (движение за кадр)
        Vector2 mouseDelta = Mouse.Delta;
        
        // Колёсико мыши
        float scroll = Input.MouseWheel;
        if ( scroll > 0 )
        {
            Log.Info( "Колёсико вверх" );
        }
        else if ( scroll < 0 )
        {
            Log.Info( "Колёсико вниз" );
        }
    }
}
```

---

## Подавление ввода

Иногда нужно **временно отключить** игровой ввод (например, когда открыт чат):

```csharp
public sealed class InputSuppressionExample : Component
{
    private bool chatOpen;
    
    protected override void OnUpdate()
    {
        // Открытие/закрытие чата
        if ( Input.Pressed( "Chat" ) )
        {
            chatOpen = !chatOpen;
        }
        
        // Подавляем ввод когда чат открыт
        // Input.Suppressed блокирует ВСЕ игровые действия
        // но мышь и клавиатура продолжают работать для UI
    }
}
```

---

## Полный пример: Контроллер с разными режимами

```csharp
/// <summary>
/// Контроллер, который переключается между режимами:
/// - FPS (стрельба)
/// - Строительство (размещение объектов)
/// - Инвентарь (выбор предметов)
/// </summary>
public sealed class MultiModeController : Component
{
    public enum ControlMode { FPS, Building, Inventory }
    
    [Property] public float MoveSpeed { get; set; } = 200f;
    [Property] public float LookSensitivity { get; set; } = 1f;
    [Property] public GameObject Camera { get; set; }
    
    public ControlMode CurrentMode { get; set; } = ControlMode.FPS;
    
    private Angles eyeAngles;
    private CharacterController cc;
    
    protected override void OnAwake()
    {
        cc = Components.Get<CharacterController>();
    }
    
    protected override void OnUpdate()
    {
        // Переключение режимов
        if ( Input.Pressed( "Menu" ) )
        {
            // Q — переключаем между FPS и строительством
            CurrentMode = CurrentMode == ControlMode.FPS 
                ? ControlMode.Building 
                : ControlMode.FPS;
            
            Log.Info( $"Режим: {CurrentMode}" );
        }
        
        // Движение (во всех режимах)
        HandleMovement();
        
        // Камера (во всех режимах)
        HandleCamera();
        
        // Действия зависят от режима
        switch ( CurrentMode )
        {
            case ControlMode.FPS:
                HandleFPSMode();
                break;
            case ControlMode.Building:
                HandleBuildingMode();
                break;
            case ControlMode.Inventory:
                HandleInventoryMode();
                break;
        }
    }
    
    void HandleMovement()
    {
        if ( cc == null ) return;
        
        float speed = Input.Down( "Run" ) ? MoveSpeed * 2 : MoveSpeed;
        var wishVel = WorldRotation * Input.AnalogMove * speed;
        
        if ( cc.IsOnGround )
        {
            cc.ApplyFriction( 5f );
            cc.Accelerate( wishVel );
            
            if ( Input.Pressed( "Jump" ) )
                cc.Punch( Vector3.Up * 300f );
        }
        else
        {
            cc.Accelerate( wishVel * 0.1f );
            cc.Velocity += Vector3.Down * 800f * Time.Delta;
        }
        
        cc.Move();
    }
    
    void HandleCamera()
    {
        var look = Input.AnalogLook;
        eyeAngles.yaw -= look.yaw * LookSensitivity;
        eyeAngles.pitch = (eyeAngles.pitch + look.pitch * LookSensitivity).Clamp( -89f, 89f );
        
        if ( Camera != null )
            Camera.WorldRotation = eyeAngles.ToRotation();
        
        WorldRotation = Rotation.FromYaw( eyeAngles.yaw );
    }
    
    void HandleFPSMode()
    {
        // ЛКМ — стрелять
        if ( Input.Pressed( "Attack1" ) )
        {
            Shoot();
        }
        
        // ПКМ — прицелиться
        if ( Input.Down( "Attack2" ) )
        {
            // Зум камеры
        }
        
        // R — перезарядка
        if ( Input.Pressed( "Reload" ) )
        {
            Log.Info( "Перезарядка!" );
        }
    }
    
    void HandleBuildingMode()
    {
        // ЛКМ — разместить блок
        if ( Input.Pressed( "Attack1" ) )
        {
            PlaceBlock();
        }
        
        // ПКМ — удалить блок
        if ( Input.Pressed( "Attack2" ) )
        {
            RemoveBlock();
        }
        
        // Колёсико — поворот блока
        if ( Input.MouseWheel != 0 )
        {
            Log.Info( $"Поворот блока: {Input.MouseWheel}" );
        }
    }
    
    void HandleInventoryMode()
    {
        // Выбор слотов
        for ( int i = 1; i <= 9; i++ )
        {
            if ( Input.Pressed( $"Slot{i}" ) )
            {
                Log.Info( $"Выбран слот {i}" );
            }
        }
    }
    
    void Shoot()
    {
        Vector3 start = Camera?.WorldPosition ?? WorldPosition;
        Vector3 end = start + eyeAngles.ToRotation().Forward * 5000;
        
        var tr = Scene.Trace.Ray( start, end )
            .IgnoreGameObject( GameObject )
            .Run();
        
        if ( tr.Hit )
        {
            var health = tr.GameObject?.Components.Get<HealthComponent>();
            health?.TakeDamage( 25 );
        }
    }
    
    void PlaceBlock() { Log.Info( "Блок размещён!" ); }
    void RemoveBlock() { Log.Info( "Блок удалён!" ); }
}
```

---

## Следующий шаг

Перейдите к разделу **[Встроенные компоненты](../09-Built-in-Components/README.md)** для полного справочника.
