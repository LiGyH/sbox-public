# 12. Анимации

## Обзор системы анимаций

s&box использует мощную систему анимаций на базе Source 2. Анимации управляются через **AnimGraph** (граф анимаций) — визуальный редактор состояний и переходов. Из кода вы задаёте **параметры**, а AnimGraph решает, какую анимацию проигрывать.

### Основные понятия

| Термин | Описание |
|--------|----------|
| **AnimGraph** | Граф анимаций — визуальная система переходов между анимациями |
| **Параметр** | Переменная в AnimGraph (float, int, bool, Vector3, Rotation), которую вы задаёте из кода |
| **Sequence** | Конкретная анимация (клип), например `idle`, `run`, `jump` |
| **IK (Inverse Kinematics)** | Система, которая подгоняет кости под целевые точки (руки к оружию, ноги к полу) |
| **SkinnedModelRenderer** | Компонент для отображения анимированных моделей |
| **Морфы** | Деформации меша (блендшейпы), например для мимики лица |

---

## SkinnedModelRenderer — основной компонент

`SkinnedModelRenderer` — это компонент для отображения **анимированных** 3D-моделей (персонажи, оружие, двери, NPC и т.д.).

### Базовая настройка

```csharp
public sealed class AnimatedCharacter : Component
{
    [Property] public SkinnedModelRenderer Renderer { get; set; }

    protected override void OnAwake()
    {
        Renderer = Components.Get<SkinnedModelRenderer>();

        // Загрузить модель
        Renderer.Model = Model.Load( "models/citizen/citizen.vmdl" );
    }
}
```

---

## Управление параметрами AnimGraph

Параметры AnimGraph — это основной способ управления анимациями. Вы задаёте значения из кода, а AnimGraph использует их для выбора анимации.

### Установка параметров

```csharp
public sealed class AnimationController : Component
{
    [Property] public SkinnedModelRenderer Renderer { get; set; }

    protected override void OnUpdate()
    {
        if ( Renderer == null ) return;

        // === FLOAT — плавные значения (скорость, высота, уровень приседания) ===
        Renderer.Set( "move_speed", 200f );
        Renderer.Set( "move_groundspeed", 200f );
        Renderer.Set( "duck", 0.5f ); // 0 = стоит, 1 = полное приседание

        // === INT — целые числа (тип оружия, состояние) ===
        Renderer.Set( "holdtype", 1 ); // 0 = без оружия, 1 = пистолет, 2 = винтовка и т.д.

        // === BOOL — включение/выключение (на земле, плавает, ползёт) ===
        Renderer.Set( "b_grounded", true );
        Renderer.Set( "b_swim", false );
        Renderer.Set( "b_climbing", false );

        // === VECTOR3 — направление (движение, прицеливание) ===
        Renderer.Set( "move_direction", Vector3.Forward );

        // === ROTATION — поворот ===
        Renderer.Set( "aim_body", Rotation.LookAt( Vector3.Forward ) );
    }
}
```

### Чтение параметров

```csharp
public sealed class ReadAnimParams : Component
{
    [Property] public SkinnedModelRenderer Renderer { get; set; }

    protected override void OnUpdate()
    {
        if ( Renderer == null ) return;

        // Чтение текущих значений параметров
        float speed = Renderer.GetFloat( "move_speed" );
        int holdType = Renderer.GetInt( "holdtype" );
        bool grounded = Renderer.GetBool( "b_grounded" );
        Vector3 moveDir = Renderer.GetVector( "move_direction" );
        Rotation aimRot = Renderer.GetRotation( "aim_body" );

        Log.Info( $"Скорость: {speed}, На земле: {grounded}" );
    }
}
```

### Управление направлением взгляда

```csharp
public sealed class LookAtController : Component
{
    [Property] public SkinnedModelRenderer Renderer { get; set; }
    [Property] public GameObject LookTarget { get; set; }

    protected override void OnUpdate()
    {
        if ( Renderer == null || LookTarget == null ) return;

        // Направление к цели
        var direction = (LookTarget.WorldPosition - WorldPosition).Normal;

        // SetLookDirection управляет отдельными каналами взгляда:
        // aim_eyes — глаза
        // aim_head — голова
        // aim_body — торс
        Renderer.SetLookDirection( "aim_eyes", direction, 1.0f );
        Renderer.SetLookDirection( "aim_head", direction, 0.8f );
        Renderer.SetLookDirection( "aim_body", direction, 0.5f );
    }
}
```

---

## Скорость воспроизведения

```csharp
public sealed class PlaybackControl : Component
{
    [Property] public SkinnedModelRenderer Renderer { get; set; }

    protected override void OnUpdate()
    {
        if ( Renderer == null ) return;

        // Скорость проигрывания анимаций (0.0 — 4.0)
        Renderer.PlaybackRate = 1.0f;   // Нормальная скорость
        // Renderer.PlaybackRate = 0.5f;  // Замедленно (slow-mo)
        // Renderer.PlaybackRate = 2.0f;  // Ускоренно
        // Renderer.PlaybackRate = 0.0f;  // Пауза (заморозка анимации)
    }
}
```

---

## Прямое воспроизведение Sequence (без AnimGraph)

Для простых случаев (анимация двери, рычага, предмета окружения) можно обойтись без AnimGraph и проигрывать анимации напрямую:

```csharp
public sealed class DirectPlayback : Component
{
    [Property] public SkinnedModelRenderer Renderer { get; set; }

    protected override void OnAwake()
    {
        if ( Renderer == null ) return;

        // Отключаем AnimGraph для прямого управления
        Renderer.UseAnimGraph = false;
    }

    public void PlayAnimation( string sequenceName, bool loop = false )
    {
        if ( Renderer == null ) return;

        // Устанавливаем последовательность
        Renderer.Sequence.Name = sequenceName;
        Renderer.Sequence.Looping = loop;
    }

    protected override void OnUpdate()
    {
        if ( Renderer == null ) return;

        // Информация о текущей анимации
        string current = Renderer.Sequence.Name;
        float duration = Renderer.Sequence.Duration;
        float time = Renderer.Sequence.Time;
        float progress = Renderer.Sequence.TimeNormalized; // 0.0 — 1.0
        bool finished = Renderer.Sequence.IsFinished;

        // Список всех доступных анимаций модели
        var allSequences = Renderer.Sequence.SequenceNames;
    }
}
```

### Пример: Анимированная дверь

```csharp
public sealed class AnimatedDoor : Component
{
    [Property] public SkinnedModelRenderer Renderer { get; set; }
    [Property] public string OpenAnimation { get; set; } = "open";
    [Property] public string CloseAnimation { get; set; } = "close";

    private bool isOpen;

    public void Toggle()
    {
        isOpen = !isOpen;

        if ( Renderer == null ) return;

        Renderer.UseAnimGraph = false;
        Renderer.Sequence.Name = isOpen ? OpenAnimation : CloseAnimation;
        Renderer.Sequence.Looping = false;
    }
}
```

### Пример: Вращающийся вентилятор (зацикленная анимация)

```csharp
public sealed class RotatingFan : Component
{
    [Property] public SkinnedModelRenderer Renderer { get; set; }
    [Property] public string SpinAnimation { get; set; } = "spin";
    [Property] public float Speed { get; set; } = 1.0f;

    protected override void OnAwake()
    {
        if ( Renderer == null ) return;

        Renderer.UseAnimGraph = false;
        Renderer.Sequence.Name = SpinAnimation;
        Renderer.Sequence.Looping = true;
        Renderer.PlaybackRate = Speed;
    }

    public void SetSpeed( float speed )
    {
        Speed = speed;
        if ( Renderer != null )
        {
            Renderer.PlaybackRate = speed;
        }
    }
}
```

---

## Inverse Kinematics (IK)

IK позволяет привязывать кости модели к целевым точкам в мире. Например, чтобы руки персонажа держали оружие или ноги стояли на наклонной поверхности.

```csharp
public sealed class IKExample : Component
{
    [Property] public SkinnedModelRenderer Renderer { get; set; }
    [Property] public GameObject LeftHandTarget { get; set; }
    [Property] public GameObject RightHandTarget { get; set; }
    [Property] public GameObject LeftFootTarget { get; set; }
    [Property] public GameObject RightFootTarget { get; set; }

    protected override void OnUpdate()
    {
        if ( Renderer == null ) return;

        // === РУКИ ===
        // Привязать левую руку к цели (например, к цевью оружия)
        if ( LeftHandTarget != null )
        {
            Renderer.SetIk( "hand_left", LeftHandTarget.WorldTransform );
        }
        else
        {
            Renderer.ClearIk( "hand_left" );
        }

        // Привязать правую руку
        if ( RightHandTarget != null )
        {
            Renderer.SetIk( "hand_right", RightHandTarget.WorldTransform );
        }
        else
        {
            Renderer.ClearIk( "hand_right" );
        }

        // === НОГИ ===
        // Привязать ноги к поверхности (для корректной постановки на неровностях)
        if ( LeftFootTarget != null )
        {
            Renderer.SetIk( "foot_left", LeftFootTarget.WorldTransform );
        }

        if ( RightFootTarget != null )
        {
            Renderer.SetIk( "foot_right", RightFootTarget.WorldTransform );
        }
    }
}
```

### Стандартные IK-цели

| Имя | Описание |
|-----|----------|
| `hand_left` | Левая рука (цевьё оружия, поручни) |
| `hand_right` | Правая рука (рукоятка оружия) |
| `foot_left` | Левая нога (подстройка под поверхность) |
| `foot_right` | Правая нога (подстройка под поверхность) |

---

## CitizenAnimationHelper — высокоуровневый хелпер

`CitizenAnimationHelper` — готовый компонент для управления анимацией модели `citizen`. Он предоставляет удобные свойства и методы вместо работы с параметрами напрямую.

### Базовое использование

```csharp
public sealed class PlayerAnimator : Component
{
    [Property] public CitizenAnimationHelper AnimHelper { get; set; }
    [Property] public CharacterController Controller { get; set; }

    protected override void OnUpdate()
    {
        if ( AnimHelper == null || Controller == null ) return;

        // Передаём скорость движения — AnimHelper сам выберет idle/walk/run
        AnimHelper.WithVelocity( Controller.Velocity );

        // Передаём желаемое направление (для анимации в воздухе)
        AnimHelper.WithWishVelocity( Input.AnalogMove * 200f );

        // На земле?
        AnimHelper.IsGrounded = Controller.IsOnGround;

        // Приседание (0 — стоит, 1 — полностью присел)
        AnimHelper.DuckLevel = Input.Down( "Duck" ) ? 1.0f : 0.0f;
    }
}
```

### Управление взглядом

```csharp
public sealed class PlayerLookAnimator : Component
{
    [Property] public CitizenAnimationHelper AnimHelper { get; set; }

    protected override void OnUpdate()
    {
        if ( AnimHelper == null ) return;

        // Куда смотрит персонаж (направление из камеры)
        var lookDir = Camera.Main.WorldRotation.Forward;

        // WithLook задаёт направление взгляда с весами для разных частей тела
        AnimHelper.WithLook(
            lookDir,
            eyesWeight: 1.0f,     // Глаза полностью следят
            headWeight: 0.8f,     // Голова почти полностью
            bodyWeight: 0.5f      // Торс наполовину
        );

        // Угол прицеливания (pitch и yaw)
        AnimHelper.AimAngle = Camera.Main.WorldRotation;
    }
}
```

### Типы удержания оружия (HoldType)

```csharp
public sealed class WeaponHoldAnimator : Component
{
    [Property] public CitizenAnimationHelper AnimHelper { get; set; }

    public void SetWeaponType( string weaponType )
    {
        if ( AnimHelper == null ) return;

        // HoldType определяет, как персонаж держит оружие
        AnimHelper.HoldType = weaponType switch
        {
            "pistol" => CitizenAnimationHelper.HoldTypes.Pistol,
            "rifle" => CitizenAnimationHelper.HoldTypes.Rifle,
            "shotgun" => CitizenAnimationHelper.HoldTypes.Shotgun,
            "smg" => CitizenAnimationHelper.HoldTypes.HoldItem,
            "rpg" => CitizenAnimationHelper.HoldTypes.Rifle,
            "melee" => CitizenAnimationHelper.HoldTypes.Swing,
            "fists" => CitizenAnimationHelper.HoldTypes.Fists,
            _ => CitizenAnimationHelper.HoldTypes.None
        };

        // Рука (для пистолета можно одной рукой)
        AnimHelper.Handedness = CitizenAnimationHelper.Hand.Right;
        // AnimHelper.Handedness = CitizenAnimationHelper.Hand.Left;
        // AnimHelper.Handedness = CitizenAnimationHelper.Hand.Both;
    }
}
```

### Триггеры — одноразовые действия

```csharp
public sealed class AnimationTriggers : Component
{
    [Property] public CitizenAnimationHelper AnimHelper { get; set; }

    // Вызовите при прыжке
    public void OnJump()
    {
        AnimHelper?.TriggerJump();
    }

    // Вызовите при экипировке/смене оружия
    public void OnWeaponDeploy()
    {
        AnimHelper?.TriggerDeploy();
    }
}
```

### Специальные анимации и состояния

```csharp
public sealed class SpecialAnimations : Component
{
    [Property] public CitizenAnimationHelper AnimHelper { get; set; }

    protected override void OnUpdate()
    {
        if ( AnimHelper == null ) return;

        // Плавание
        AnimHelper.IsSwimming = false;

        // Лазание по лестнице
        AnimHelper.IsClimbing = false;

        // Noclip (полёт без коллизий)
        AnimHelper.IsNoclipping = false;

        // Сидение
        AnimHelper.Sitting = CitizenAnimationHelper.SittingStyle.None;
        // AnimHelper.Sitting = CitizenAnimationHelper.SittingStyle.Chair;
        // AnimHelper.Sitting = CitizenAnimationHelper.SittingStyle.Floor;

        // Специальные движения
        AnimHelper.SpecialMove = CitizenAnimationHelper.SpecialMoveStyle.None;
        // AnimHelper.SpecialMove = CitizenAnimationHelper.SpecialMoveStyle.LedgeGrab;
        // AnimHelper.SpecialMove = CitizenAnimationHelper.SpecialMoveStyle.Roll;
        // AnimHelper.SpecialMove = CitizenAnimationHelper.SpecialMoveStyle.Slide;

        // Уровень голоса (для анимации рта при разговоре)
        AnimHelper.VoiceLevel = 0.0f;
    }
}
```

### Реакция на урон (процедурная)

```csharp
public sealed class DamageReaction : Component
{
    [Property] public CitizenAnimationHelper AnimHelper { get; set; }

    public void OnTakeDamage( DamageInfo info )
    {
        if ( AnimHelper == null ) return;

        // Процедурная реакция на урон — персонаж дёргается от попадания
        AnimHelper.ProceduralHitReaction(
            info,
            damageScale: 1.0f   // Масштаб реакции (больше = сильнее отклонение)
        );
    }
}
```

---

## Анимации объектов окружения

Анимировать можно не только персонажей. Любой объект с `SkinnedModelRenderer` может быть анимированным.

### Примеры объектов окружения

```csharp
/// <summary>
/// Простой переключатель (рычаг, кнопка) с анимацией.
/// </summary>
public sealed class AnimatedSwitch : Component
{
    [Property] public SkinnedModelRenderer Renderer { get; set; }
    [Property] public string OnAnimation { get; set; } = "on";
    [Property] public string OffAnimation { get; set; } = "off";

    private bool isOn;

    public void Toggle()
    {
        isOn = !isOn;

        if ( Renderer == null ) return;

        // Для простых объектов — прямой Sequence
        Renderer.UseAnimGraph = false;
        Renderer.Sequence.Name = isOn ? OnAnimation : OffAnimation;
        Renderer.Sequence.Looping = false;
    }
}
```

```csharp
/// <summary>
/// Конвейерная лента — зацикленная анимация.
/// </summary>
public sealed class ConveyorBelt : Component
{
    [Property] public SkinnedModelRenderer Renderer { get; set; }
    [Property] public float Speed { get; set; } = 1.0f;
    [Property] public bool IsRunning { get; set; } = true;

    protected override void OnAwake()
    {
        if ( Renderer == null ) return;

        Renderer.UseAnimGraph = false;
        Renderer.Sequence.Name = "move";
        Renderer.Sequence.Looping = true;
    }

    protected override void OnUpdate()
    {
        if ( Renderer == null ) return;

        // Останавливаем/запускаем через скорость воспроизведения
        Renderer.PlaybackRate = IsRunning ? Speed : 0f;
    }
}
```

---

## Полный пример: Анимированный персонаж

```csharp
/// <summary>
/// Полная настройка анимации персонажа от первого/третьего лица.
/// </summary>
public sealed class FullCharacterAnimator : Component
{
    [Property] public SkinnedModelRenderer BodyRenderer { get; set; }
    [Property] public CitizenAnimationHelper AnimHelper { get; set; }
    [Property] public CharacterController Controller { get; set; }
    [Property] public GameObject CameraObject { get; set; }

    [Sync] public Angles EyeAngles { get; set; }

    protected override void OnUpdate()
    {
        if ( IsProxy )
        {
            // Для чужих игроков — только обновляем взгляд
            UpdateLook();
            return;
        }

        HandleCamera();
        UpdateAnimation();
    }

    void HandleCamera()
    {
        var look = Input.AnalogLook;
        var angles = EyeAngles;
        angles.yaw -= look.yaw;
        angles.pitch += look.pitch;
        angles.pitch = angles.pitch.Clamp( -89f, 89f );
        EyeAngles = angles;
    }

    void UpdateAnimation()
    {
        if ( AnimHelper == null || Controller == null ) return;

        // Скорость и направление
        AnimHelper.WithVelocity( Controller.Velocity );
        AnimHelper.WithWishVelocity( Input.AnalogMove * 200f );

        // Состояние
        AnimHelper.IsGrounded = Controller.IsOnGround;
        AnimHelper.DuckLevel = Input.Down( "Duck" ) ? 1.0f : 0.0f;

        // Взгляд
        UpdateLook();

        // Прыжок
        if ( Input.Pressed( "Jump" ) && Controller.IsOnGround )
        {
            AnimHelper.TriggerJump();
        }
    }

    void UpdateLook()
    {
        if ( AnimHelper == null ) return;

        AnimHelper.AimAngle = EyeAngles.ToRotation();

        var lookDir = EyeAngles.ToRotation().Forward;
        AnimHelper.WithLook( lookDir, 1f, 0.8f, 0.5f );
    }
}
```

---

## Таблица основных параметров AnimGraph

| Параметр | Тип | Описание |
|----------|-----|----------|
| `move_speed` | float | Скорость передвижения |
| `move_groundspeed` | float | Скорость по земле |
| `move_direction` | Vector3 | Направление движения |
| `duck` | float | Уровень приседания (0-1) |
| `holdtype` | int | Тип удержания оружия |
| `b_grounded` | bool | На земле |
| `b_swim` | bool | Плавает |
| `b_climbing` | bool | Ползёт/лезет |
| `b_sit` | bool | Сидит |
| `aim_eyes` | Rotation | Направление глаз |
| `aim_head` | Rotation | Направление головы |
| `aim_body` | Rotation | Направление торса |
| `voice` | float | Уровень голоса (0-1) |

---

## Следующий шаг

Перейдите к разделу **[Оружие: WorldModel, ViewModel и Prediction](../13-Weapons-Advanced/README.md)** для изучения продвинутой работы с оружием.
