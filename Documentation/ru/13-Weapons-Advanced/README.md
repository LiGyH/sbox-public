# 13. Оружие: WorldModel, ViewModel и Prediction

## Обзор

В разделе [DarkRP: Оружие](../11-DarkRP-Example/07-Weapons.md) был показан базовый пример оружия. Здесь мы рассмотрим **продвинутые** аспекты:

- **WorldModel** vs **ViewModel** — в чём разница и зачем два типа модели
- **Анимации оружия** — для каждого типа модели
- **Prediction (предсказание)** — как сделать стрельбу отзывчивой в мультиплеере
- **Слой рендеринга** — как ViewModel рисуется поверх мира

---

## WorldModel vs ViewModel

### Что такое WorldModel?

**WorldModel (мировая модель)** — это модель оружия, которую видят **другие игроки**. Она отображается в мире от третьего лица.

- Видна **всем** игрокам
- Прикрепляется к костям рук персонажа (через IK или attachment)
- Обычно имеет **менее детализированную** геометрию
- Файл модели обычно без префикса: `w_pistol.vmdl` или `pistol.vmdl`

### Что такое ViewModel?

**ViewModel (модель от первого лица)** — это модель оружия (часто с руками), которую видит **только владелец** оружия.

- Видна **только** владельцу (от первого лица)
- Рисуется на отдельном слое поверх всего мира
- Обычно **очень детализированная** (игрок видит её вблизи)
- Включает модель рук, анимации перезарядки, прицеливания
- Файл модели обычно с префиксом `v_`: `v_pistol.vmdl`

### Визуальная схема

```
Игрок 1 (Владелец оружия)        Игрок 2 (Другой игрок)
┌─────────────────────┐           ┌─────────────────────┐
│                     │           │                     │
│   Видит ViewModel   │           │ Видит WorldModel    │
│   (руки + оружие    │           │ (оружие в руках     │
│    крупным планом)  │           │  модели персонажа)  │
│                     │           │                     │
└─────────────────────┘           └─────────────────────┘
```

---

## Слой рендеринга ViewModel

ViewModel рисуется на специальном слое `SceneRenderLayer.ViewModel`, чтобы оружие **не проваливалось** в стены и другие объекты.

### SceneRenderLayer

```csharp
// Доступные слои рендеринга
public enum SceneRenderLayer
{
    Default = 0,              // Обычный мир (WorldModel и всё остальное)
    ViewModel = 10,           // Поверх мира с изменённой глубиной
    OverlayWithDepth = 20,    // После пост-обработки, с глубиной сцены
    OverlayWithoutDepth = 30  // После пост-обработки, без глубины
}
```

### Настройка ViewModel

```csharp
public sealed class WeaponViewModel : Component
{
    [Property] public SkinnedModelRenderer ViewModelRenderer { get; set; }

    protected override void OnAwake()
    {
        if ( ViewModelRenderer == null ) return;

        // Устанавливаем слой рендеринга ViewModel
        // Это делает модель видимой ТОЛЬКО для камеры владельца
        // и рисует её ПОВЕРХ мира
        ViewModelRenderer.RenderType = ModelRenderer.ShadowRenderType.Off;
    }
}
```

---

## Полная архитектура оружия

### Структура GameObject оружия

```
PlayerObject
├── Body (SkinnedModelRenderer — модель персонажа)
├── Camera
│   └── ViewModel (SkinnedModelRenderer — v_pistol.vmdl)
│       └── Слой: SceneRenderLayer.ViewModel
└── WeaponWorldModel (SkinnedModelRenderer — w_pistol.vmdl)
    └── Прикреплён к руке персонажа через IK/attachment
```

### Компонент оружия с двумя моделями

```csharp
/// <summary>
/// Продвинутое оружие с WorldModel и ViewModel.
/// </summary>
public sealed class AdvancedWeapon : Component
{
    // === МОДЕЛИ ===
    [Property, Group( "Модели" )] public Model WorldModelAsset { get; set; }
    [Property, Group( "Модели" )] public Model ViewModelAsset { get; set; }

    // === РЕНДЕРЕРЫ ===
    [Property, Group( "Ссылки" )] public SkinnedModelRenderer WorldModelRenderer { get; set; }
    [Property, Group( "Ссылки" )] public SkinnedModelRenderer ViewModelRenderer { get; set; }
    [Property, Group( "Ссылки" )] public CitizenAnimationHelper AnimHelper { get; set; }

    // === НАСТРОЙКИ ===
    [Property, Group( "Оружие" )] public float Damage { get; set; } = 25f;
    [Property, Group( "Оружие" )] public float FireRate { get; set; } = 0.15f;
    [Property, Group( "Оружие" )] public int MagazineSize { get; set; } = 12;

    // === СЕТЕВЫЕ ===
    [Sync] public int CurrentAmmo { get; set; }
    [Sync] public bool IsReloading { get; set; }
    [Sync] public bool IsAiming { get; set; }

    private TimeUntil nextShot;

    protected override void OnAwake()
    {
        CurrentAmmo = MagazineSize;
        SetupModels();
    }

    void SetupModels()
    {
        // WorldModel — видна всем
        if ( WorldModelRenderer != null && WorldModelAsset != null )
        {
            WorldModelRenderer.Model = WorldModelAsset;
        }

        // ViewModel — видна только владельцу
        if ( ViewModelRenderer != null && ViewModelAsset != null )
        {
            ViewModelRenderer.Model = ViewModelAsset;
        }
    }

    protected override void OnUpdate()
    {
        if ( IsProxy )
        {
            // Для других игроков: только обновляем WorldModel
            UpdateWorldModelVisibility();
            return;
        }

        // Для владельца: управление + обе модели
        UpdateViewModelVisibility();
        UpdateWorldModelVisibility();
        HandleInput();
    }

    void UpdateViewModelVisibility()
    {
        // ViewModel видна ТОЛЬКО владельцу
        if ( ViewModelRenderer != null )
        {
            ViewModelRenderer.Enabled = !IsProxy;
        }
    }

    void UpdateWorldModelVisibility()
    {
        // WorldModel видна ТОЛЬКО другим игрокам
        if ( WorldModelRenderer != null )
        {
            WorldModelRenderer.Enabled = IsProxy;
        }
    }

    void HandleInput()
    {
        IsAiming = Input.Down( "Attack2" );

        if ( Input.Pressed( "Attack1" ) || (Input.Down( "Attack1" ) && nextShot <= 0) )
        {
            TryShoot();
        }

        if ( Input.Pressed( "Reload" ) )
        {
            StartReload();
        }
    }

    void TryShoot()
    {
        if ( IsReloading || nextShot > 0 || CurrentAmmo <= 0 ) return;

        CurrentAmmo--;
        nextShot = FireRate;

        // Анимация ViewModel (для владельца)
        PlayViewModelAnimation( "fire" );

        // Анимация WorldModel (для остальных)
        BroadcastShootEffects();
    }

    void PlayViewModelAnimation( string anim )
    {
        if ( ViewModelRenderer == null ) return;

        // Триггер анимации на ViewModel
        ViewModelRenderer.Set( "b_attack", true );
    }

    [Broadcast]
    void BroadcastShootEffects()
    {
        // WorldModel: анимация стрельбы на персонаже
        if ( WorldModelRenderer != null )
        {
            WorldModelRenderer.Set( "b_attack", true );
        }

        // Звук, частицы и т.д. (все клиенты)
    }

    [Broadcast]
    void StartReload()
    {
        IsReloading = true;

        // Анимация перезарядки: ViewModel
        if ( ViewModelRenderer != null )
        {
            ViewModelRenderer.Set( "b_reload", true );
        }

        // Анимация перезарядки: WorldModel
        if ( WorldModelRenderer != null )
        {
            WorldModelRenderer.Set( "b_reload", true );
        }
    }
}
```

---

## Анимации оружия

### Анимации ViewModel (от первого лица)

ViewModel обычно имеет собственный AnimGraph со следующими параметрами:

```csharp
public sealed class ViewModelAnimator : Component
{
    [Property] public SkinnedModelRenderer ViewModelRenderer { get; set; }

    public void PlayIdle()
    {
        ViewModelRenderer?.Set( "b_deploy", false );
    }

    public void PlayDeploy()
    {
        // Анимация доставания оружия
        ViewModelRenderer?.Set( "b_deploy", true );
    }

    public void PlayAttack()
    {
        // Анимация выстрела / удара
        ViewModelRenderer?.Set( "b_attack", true );
    }

    public void PlayReload()
    {
        // Анимация перезарядки
        ViewModelRenderer?.Set( "b_reload", true );
    }

    public void SetAiming( bool aiming )
    {
        // Прицеливание (переход к ADS - aim down sights)
        ViewModelRenderer?.Set( "b_aim", aiming );
    }

    public void SetEmpty( bool empty )
    {
        // Пустой магазин (затвор открыт)
        ViewModelRenderer?.Set( "b_empty", empty );
    }
}
```

### Анимации WorldModel (для персонажа)

WorldModel управляется через `CitizenAnimationHelper` на персонаже:

```csharp
public sealed class WeaponWorldAnimator : Component
{
    [Property] public CitizenAnimationHelper AnimHelper { get; set; }

    public void SetHoldType( string weaponType )
    {
        if ( AnimHelper == null ) return;

        // Тип удержания определяет позу персонажа с оружием
        AnimHelper.HoldType = weaponType switch
        {
            "pistol" => CitizenAnimationHelper.HoldTypes.Pistol,
            "rifle" => CitizenAnimationHelper.HoldTypes.Rifle,
            "shotgun" => CitizenAnimationHelper.HoldTypes.Shotgun,
            "melee" => CitizenAnimationHelper.HoldTypes.Swing,
            "fists" => CitizenAnimationHelper.HoldTypes.Fists,
            _ => CitizenAnimationHelper.HoldTypes.None
        };
    }

    public void OnDeploy()
    {
        // Триггер анимации доставания на модели персонажа
        AnimHelper?.TriggerDeploy();
    }
}
```

---

## Prediction (Предсказание)

### Проблема: задержка в мультиплеере

В мультиплеере между нажатием кнопки и ответом сервера есть **задержка** (пинг). Без предсказания:

```
Клиент: Нажал "Стрельба"  →  [50мс задержка]  →  Сервер: Обработал
                                                         ↓
Клиент: Увидел результат  ←  [50мс задержка]  ←  Сервер: Отправил результат

Итого: ~100мс задержки — игрок чувствует "тормоза"
```

### Решение: клиентское предсказание

Предсказание позволяет клиенту **немедленно** выполнить действие, не дожидаясь сервера:

```
Клиент: Нажал "Стрельба" → СРАЗУ проигрывает анимацию и звук
                          → СРАЗУ уменьшает патроны в UI
                          → Отправляет запрос серверу

Сервер: Проверяет и подтверждает (или откатывает, если нарушение)
```

### Реализация предсказания в s&box

В s&box предсказание реализуется через паттерн **"делай локально, синхронизируй по сети"**:

```csharp
/// <summary>
/// Оружие с клиентским предсказанием.
/// Стрельба, перезарядка и эффекты происходят мгновенно на клиенте.
/// </summary>
public sealed class PredictedWeapon : Component
{
    [Property] public float Damage { get; set; } = 25f;
    [Property] public float FireRate { get; set; } = 0.15f;
    [Property] public int MagazineSize { get; set; } = 30;

    [Property] public SkinnedModelRenderer ViewModelRenderer { get; set; }
    [Property] public SoundEvent ShootSound { get; set; }

    // Синхронизируемые свойства (авторитетные данные)
    [Sync] public int CurrentAmmo { get; set; }
    [Sync] public bool IsReloading { get; set; }

    private TimeUntil nextShot;
    private TimeUntil reloadFinish;

    protected override void OnAwake()
    {
        CurrentAmmo = MagazineSize;
    }

    protected override void OnUpdate()
    {
        // Только владелец обрабатывает ввод
        if ( IsProxy ) return;

        if ( Input.Pressed( "Attack1" ) || Input.Down( "Attack1" ) )
        {
            TryShoot();
        }

        if ( Input.Pressed( "Reload" ) )
        {
            TryReload();
        }

        if ( IsReloading && reloadFinish <= 0 )
        {
            FinishReload();
        }
    }

    void TryShoot()
    {
        if ( IsReloading || nextShot > 0 || CurrentAmmo <= 0 ) return;

        // === ПРЕДСКАЗАНИЕ: Немедленно на клиенте ===

        CurrentAmmo--;
        nextShot = FireRate;

        // Локальный эффект — мгновенный отклик
        PlayLocalShootEffects();

        // Сетевой эффект — для остальных игроков
        BroadcastShootEffects();

        // Стрельба (трассировка)
        PerformShot();
    }

    void PlayLocalShootEffects()
    {
        // ViewModel анимация — ТОЛЬКО для владельца, МГНОВЕННО
        ViewModelRenderer?.Set( "b_attack", true );

        // Локальный звук — МГНОВЕННО
        if ( ShootSound != null )
        {
            Sound.Play( ShootSound, WorldPosition );
        }
    }

    [Broadcast]
    void BroadcastShootEffects()
    {
        // Это выполняется на ВСЕХ клиентах
        // Для владельца: уже проиграно локально, поэтому проверяем
        if ( !IsProxy ) return; // Владелец уже проиграл эффекты

        // Звук и эффекты для ДРУГИХ игроков
        if ( ShootSound != null )
        {
            Sound.Play( ShootSound, WorldPosition );
        }
    }

    void PerformShot()
    {
        var eyePos = Scene.Camera?.WorldPosition ?? WorldPosition;
        var eyeDir = Scene.Camera?.WorldRotation.Forward ?? WorldRotation.Forward;

        var tr = Scene.Trace
            .Ray( eyePos, eyePos + eyeDir * 5000f )
            .IgnoreGameObject( GameObject )
            .WithoutTags( "trigger" )
            .Run();

        if ( tr.Hit && tr.GameObject != null )
        {
            var damageable = tr.GameObject.Components.GetInParent<IDamageable>();
            if ( damageable != null )
            {
                var info = new DamageInfo( Damage, GameObject, GameObject );
                damageable.OnDamage( info );
            }
        }
    }

    void TryReload()
    {
        if ( IsReloading || CurrentAmmo >= MagazineSize ) return;

        IsReloading = true;
        reloadFinish = 2.0f;

        // ViewModel анимация перезарядки — МГНОВЕННО
        ViewModelRenderer?.Set( "b_reload", true );

        BroadcastReloadEffects();
    }

    [Broadcast]
    void BroadcastReloadEffects()
    {
        if ( !IsProxy ) return;
        // Звук перезарядки для других игроков
    }

    void FinishReload()
    {
        IsReloading = false;
        CurrentAmmo = MagazineSize;
    }
}
```

### Ключевые принципы предсказания

| Принцип | Описание |
|---------|----------|
| **Локальные эффекты** | Звук, анимация ViewModel, частицы — проигрываются мгновенно на клиенте |
| **Broadcast для остальных** | Сетевые эффекты отправляются через `[Broadcast]` для других игроков |
| **[Sync] для данных** | Авторитетные данные (патроны, здоровье) синхронизируются через `[Sync]` |
| **IsProxy проверка** | `if (IsProxy) return;` — не обрабатываем ввод на чужих объектах |
| **Дублирование эффектов** | В `[Broadcast]` проверяем `if (!IsProxy) return;` чтобы владелец не проигрывал эффект дважды |

---

## Переключение оружия

```csharp
/// <summary>
/// Система инвентаря оружия с переключением.
/// </summary>
public sealed class WeaponInventory : Component
{
    [Property] public List<GameObject> Weapons { get; set; } = new();
    [Sync] public int ActiveWeaponIndex { get; set; } = 0;

    protected override void OnUpdate()
    {
        if ( IsProxy ) return;

        // Переключение колесом мыши
        int scroll = (int)Input.MouseWheel.y;
        if ( scroll != 0 )
        {
            SwitchWeapon( ActiveWeaponIndex + scroll );
        }

        // Переключение цифрами
        for ( int i = 0; i < Weapons.Count && i < 9; i++ )
        {
            if ( Input.Pressed( $"Slot{i + 1}" ) )
            {
                SwitchWeapon( i );
            }
        }
    }

    void SwitchWeapon( int index )
    {
        if ( Weapons.Count == 0 ) return;

        // Оборачиваем индекс
        index = ((index % Weapons.Count) + Weapons.Count) % Weapons.Count;

        if ( index == ActiveWeaponIndex ) return;

        ActiveWeaponIndex = index;
        BroadcastWeaponSwitch( index );
    }

    [Broadcast]
    void BroadcastWeaponSwitch( int index )
    {
        // Деактивируем все оружия
        for ( int i = 0; i < Weapons.Count; i++ )
        {
            if ( Weapons[i] != null )
            {
                Weapons[i].Enabled = (i == index);
            }
        }
    }
}
```

---

## Сравнительная таблица WorldModel vs ViewModel

| Свойство | WorldModel | ViewModel |
|----------|-----------|-----------|
| **Кто видит** | Все другие игроки | Только владелец |
| **Слой рендеринга** | Default (0) | ViewModel (10) |
| **Детализация** | Низкая/средняя | Высокая |
| **Анимации** | Через CitizenAnimationHelper | Свой AnimGraph |
| **Включает руки** | Нет (руки персонажа) | Да (руки часть модели) |
| **Прикрепление** | К костям персонажа | К камере игрока |
| **Проходит сквозь стены** | Да (обычная геометрия) | Нет (свой depth buffer) |
| **Файл модели** | `w_pistol.vmdl` | `v_pistol.vmdl` |

---

## Следующий шаг

Перейдите к разделу **[Морфы и Faceposer](../14-Morphs-Faceposer/README.md)** для изучения системы лицевой анимации.
