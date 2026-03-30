# 14. Морфы и Faceposer

## Что такое морфы?

**Морфы (Morphs)** — это **деформации** меша модели, также известные как **блендшейпы** (blend shapes) или **flex controllers**. Они позволяют плавно изменять форму 3D-модели без костей.

### Основные применения

| Применение | Описание |
|-----------|----------|
| **Мимика лица** | Улыбка, хмурость, моргание, поднятие бровей |
| **Виземы (Visemes)** | Формы рта для синхронизации с речью |
| **Деформации тела** | Изменение телосложения (толщина, мускулатура) |
| **Повреждения** | Раны, царапины, вмятины на моделях |
| **Стилизация** | Мультяшные выражения, эмоции |

---

## Model.Morphs — информация о морфах модели

Каждая модель содержит **список доступных морфов**. Вы можете получить их через API:

```csharp
public sealed class MorphInfo : Component
{
    [Property] public SkinnedModelRenderer Renderer { get; set; }

    protected override void OnAwake()
    {
        if ( Renderer?.Model == null ) return;

        var morphs = Renderer.Model.Morphs;

        // Количество морфов в модели
        int count = morphs.Count;
        Log.Info( $"Модель имеет {count} морфов" );

        // Все имена морфов
        string[] names = morphs.Names;
        foreach ( var name in names )
        {
            Log.Info( $"  Морф: {name}" );
        }

        // Получить имя по индексу
        string firstName = morphs.GetName( 0 );

        // Получить индекс по имени
        int index = morphs.GetIndex( "smile" );
    }
}
```

---

## Управление морфами через MorphAccessor

`SkinnedModelRenderer` предоставляет доступ к морфам через свойство `Morphs`:

### Установка значения морфа

```csharp
public sealed class MorphControl : Component
{
    [Property] public SkinnedModelRenderer Renderer { get; set; }

    protected override void OnUpdate()
    {
        if ( Renderer == null ) return;

        // Установить значение морфа (0.0 — 1.0)
        Renderer.Morphs.Set( "smile", 0.8f );           // Улыбка на 80%
        Renderer.Morphs.Set( "blink_left", 1.0f );      // Левый глаз закрыт
        Renderer.Morphs.Set( "blink_right", 1.0f );     // Правый глаз закрыт

        // Установить с плавным переходом (fadeTime в секундах)
        Renderer.Morphs.Set( "anger", 0.5f, 0.3f );     // Переход за 0.3 сек

        // Чтение текущего значения
        float smileValue = Renderer.Morphs.Get( "smile" );

        // Сброс морфа (возврат к 0)
        Renderer.Morphs.Clear( "smile" );                // Мгновенный сброс
        Renderer.Morphs.Clear( "anger", 0.5f );          // Плавный сброс за 0.5 сек

        // Проверка, есть ли переопределение
        bool hasOverride = Renderer.Morphs.ContainsOverride( "smile" );

        // Список всех доступных морфов
        string[] allMorphs = Renderer.Morphs.Names;
    }
}
```

---

## Практические примеры

### Пример 1: Моргание

```csharp
/// <summary>
/// Автоматическое моргание персонажа.
/// </summary>
public sealed class BlinkController : Component
{
    [Property] public SkinnedModelRenderer Renderer { get; set; }
    [Property] public float BlinkInterval { get; set; } = 3.5f;
    [Property] public float BlinkDuration { get; set; } = 0.15f;

    private TimeUntil nextBlink;
    private TimeUntil blinkEnd;
    private bool isBlinking;

    protected override void OnAwake()
    {
        nextBlink = BlinkInterval + Random.Shared.Float( -1f, 1f );
    }

    protected override void OnUpdate()
    {
        if ( Renderer == null ) return;

        if ( !isBlinking && nextBlink <= 0 )
        {
            // Начинаем моргание
            isBlinking = true;
            blinkEnd = BlinkDuration;

            Renderer.Morphs.Set( "blink_left", 1.0f, 0.05f );
            Renderer.Morphs.Set( "blink_right", 1.0f, 0.05f );
        }

        if ( isBlinking && blinkEnd <= 0 )
        {
            // Заканчиваем моргание
            isBlinking = false;
            nextBlink = BlinkInterval + Random.Shared.Float( -1f, 1f );

            Renderer.Morphs.Clear( "blink_left", 0.05f );
            Renderer.Morphs.Clear( "blink_right", 0.05f );
        }
    }
}
```

### Пример 2: Эмоции персонажа

```csharp
/// <summary>
/// Система эмоций через морфы.
/// </summary>
public sealed class EmotionController : Component
{
    [Property] public SkinnedModelRenderer Renderer { get; set; }

    public enum Emotion
    {
        Neutral,
        Happy,
        Sad,
        Angry,
        Surprised
    }

    [Sync] public Emotion CurrentEmotion { get; set; } = Emotion.Neutral;

    private Emotion lastEmotion = Emotion.Neutral;

    protected override void OnUpdate()
    {
        if ( Renderer == null ) return;
        if ( CurrentEmotion == lastEmotion ) return;

        // Сбрасываем все эмоции
        ResetAllEmotions();

        // Устанавливаем новую эмоцию
        float fadeTime = 0.3f;

        switch ( CurrentEmotion )
        {
            case Emotion.Happy:
                Renderer.Morphs.Set( "smile", 0.8f, fadeTime );
                Renderer.Morphs.Set( "cheek_raise", 0.5f, fadeTime );
                break;

            case Emotion.Sad:
                Renderer.Morphs.Set( "frown", 0.7f, fadeTime );
                Renderer.Morphs.Set( "brow_inner_up", 0.6f, fadeTime );
                break;

            case Emotion.Angry:
                Renderer.Morphs.Set( "brow_down_left", 0.8f, fadeTime );
                Renderer.Morphs.Set( "brow_down_right", 0.8f, fadeTime );
                Renderer.Morphs.Set( "lip_tightener", 0.5f, fadeTime );
                break;

            case Emotion.Surprised:
                Renderer.Morphs.Set( "brow_inner_up", 0.9f, fadeTime );
                Renderer.Morphs.Set( "brow_outer_up", 0.8f, fadeTime );
                Renderer.Morphs.Set( "jaw_open", 0.4f, fadeTime );
                break;
        }

        lastEmotion = CurrentEmotion;
    }

    void ResetAllEmotions()
    {
        float fadeTime = 0.2f;
        Renderer.Morphs.Clear( "smile", fadeTime );
        Renderer.Morphs.Clear( "cheek_raise", fadeTime );
        Renderer.Morphs.Clear( "frown", fadeTime );
        Renderer.Morphs.Clear( "brow_inner_up", fadeTime );
        Renderer.Morphs.Clear( "brow_down_left", fadeTime );
        Renderer.Morphs.Clear( "brow_down_right", fadeTime );
        Renderer.Morphs.Clear( "brow_outer_up", fadeTime );
        Renderer.Morphs.Clear( "lip_tightener", fadeTime );
        Renderer.Morphs.Clear( "jaw_open", fadeTime );
    }
}
```

### Пример 3: Интерактивные морфы (редактор персонажа)

```csharp
/// <summary>
/// Редактор внешности через слайдеры морфов.
/// </summary>
public sealed class CharacterCustomizer : Component
{
    [Property] public SkinnedModelRenderer Renderer { get; set; }

    // Настройки внешности (0-1)
    [Property, Range( 0, 1 )] public float FaceWidth { get; set; } = 0.5f;
    [Property, Range( 0, 1 )] public float NoseSize { get; set; } = 0.5f;
    [Property, Range( 0, 1 )] public float JawWidth { get; set; } = 0.5f;
    [Property, Range( 0, 1 )] public float CheekBone { get; set; } = 0.5f;
    [Property, Range( 0, 1 )] public float EyeSize { get; set; } = 0.5f;

    protected override void OnUpdate()
    {
        if ( Renderer == null ) return;

        // Применяем морфы на основе настроек
        Renderer.Morphs.Set( "face_width", FaceWidth );
        Renderer.Morphs.Set( "nose_size", NoseSize );
        Renderer.Morphs.Set( "jaw_width", JawWidth );
        Renderer.Morphs.Set( "cheekbone", CheekBone );
        Renderer.Morphs.Set( "eye_size", EyeSize );
    }
}
```

---

## Виземы (Visemes) — синхронизация губ с речью

**Виземы** — это формы рта, соответствующие определённым звукам речи. Они используются для **липсинка** (lip sync) — анимации губ при разговоре.

### Стандартные виземы

| Визема | Звук | Описание |
|--------|------|----------|
| `viseme_aa` | «а» | Широко открытый рот |
| `viseme_ee` | «и» | Растянутые губы |
| `viseme_ih` | «э» | Слегка открытый рот |
| `viseme_oh` | «о» | Округлённые губы |
| `viseme_ou` | «у» | Вытянутые губы |
| `viseme_pp` | «п», «б», «м» | Сжатые губы |
| `viseme_ff` | «ф», «в» | Нижняя губа к зубам |
| `viseme_th` | «т», «д» | Язык к зубам |
| `viseme_ss` | «с», «з» | Зубы сжаты |
| `viseme_ch` | «ч», «ш» | Округлённые, слегка открытые |
| `viseme_rr` | «р» | Слегка открытый рот |
| `viseme_nn` | «н» | Слегка открытый, язык к нёбу |
| `viseme_sil` | Тишина | Закрытый рот |

### Пример: Простой липсинк через голосовой уровень

```csharp
/// <summary>
/// Простая анимация рта на основе громкости голоса.
/// </summary>
public sealed class SimpleLipSync : Component
{
    [Property] public SkinnedModelRenderer Renderer { get; set; }
    [Property] public CitizenAnimationHelper AnimHelper { get; set; }

    /// <summary>
    /// Уровень голоса (0-1). Передайте громкость микрофона или аудио.
    /// </summary>
    [Sync] public float VoiceLevel { get; set; }

    protected override void OnUpdate()
    {
        if ( Renderer == null ) return;

        // Вариант 1: Через CitizenAnimationHelper
        if ( AnimHelper != null )
        {
            AnimHelper.VoiceLevel = VoiceLevel;
        }

        // Вариант 2: Напрямую через морфы
        // Простое открытие рта пропорционально громкости
        Renderer.Morphs.Set( "jaw_open", VoiceLevel * 0.6f );
        Renderer.Morphs.Set( "viseme_aa", VoiceLevel * 0.4f );
    }
}
```

### Пример: Продвинутый липсинк с виземами

```csharp
/// <summary>
/// Продвинутая система липсинка с набором визем.
/// </summary>
public sealed class AdvancedLipSync : Component
{
    [Property] public SkinnedModelRenderer Renderer { get; set; }

    private string[] visemeNames = new[]
    {
        "viseme_sil", "viseme_pp", "viseme_ff", "viseme_th",
        "viseme_ss", "viseme_ch", "viseme_rr", "viseme_nn",
        "viseme_aa", "viseme_ee", "viseme_ih", "viseme_oh", "viseme_ou"
    };

    /// <summary>
    /// Установить активную визему по индексу с весом.
    /// </summary>
    public void SetViseme( int index, float weight )
    {
        if ( Renderer == null ) return;

        float fadeTime = 0.05f; // Быстрый переход

        for ( int i = 0; i < visemeNames.Length; i++ )
        {
            if ( i == index )
            {
                Renderer.Morphs.Set( visemeNames[i], weight, fadeTime );
            }
            else
            {
                Renderer.Morphs.Clear( visemeNames[i], fadeTime );
            }
        }
    }

    /// <summary>
    /// Установить смешение нескольких визем.
    /// </summary>
    public void SetVisemeBlend( Dictionary<string, float> visemeWeights )
    {
        if ( Renderer == null ) return;

        float fadeTime = 0.05f;

        foreach ( var visemeName in visemeNames )
        {
            if ( visemeWeights.TryGetValue( visemeName, out float weight ) )
            {
                Renderer.Morphs.Set( visemeName, weight, fadeTime );
            }
            else
            {
                Renderer.Morphs.Clear( visemeName, fadeTime );
            }
        }
    }
}
```

---

## Faceposer — инструмент лицевой анимации

**Faceposer** — это общее название для инструмента/системы создания лицевых анимаций. В s&box он реализован через комбинацию **морфов** и **анимационных параметров**.

### Принцип работы

1. **Модель** содержит набор морфов (блендшейпов) лица
2. **Код** управляет значениями морфов в реальном времени
3. **Виземы** синхронизируют рот со звуком
4. **AnimGraph** может управлять морфами автоматически (через параметр `voice`)

### Полный пример: Faceposer-контроллер

```csharp
/// <summary>
/// Полный контроллер лицевой анимации.
/// Управляет морганием, эмоциями, взглядом и липсинком.
/// </summary>
public sealed class FaceposerController : Component
{
    [Property] public SkinnedModelRenderer Renderer { get; set; }
    [Property] public CitizenAnimationHelper AnimHelper { get; set; }

    // === Настройки ===
    [Property, Group( "Моргание" )] public float BlinkInterval { get; set; } = 4f;
    [Property, Group( "Моргание" )] public float BlinkSpeed { get; set; } = 0.1f;

    // === Состояние ===
    [Sync] public float VoiceLevel { get; set; }
    [Sync] public int EmotionIndex { get; set; }

    private TimeUntil nextBlink;
    private bool isBlinking;

    protected override void OnAwake()
    {
        nextBlink = BlinkInterval;
    }

    protected override void OnUpdate()
    {
        if ( Renderer == null ) return;

        UpdateBlink();
        UpdateLipSync();
        UpdateEmotion();
    }

    void UpdateBlink()
    {
        if ( nextBlink <= 0 && !isBlinking )
        {
            isBlinking = true;
            Renderer.Morphs.Set( "blink_left", 1f, BlinkSpeed );
            Renderer.Morphs.Set( "blink_right", 1f, BlinkSpeed );
        }

        if ( isBlinking )
        {
            float val = Renderer.Morphs.Get( "blink_left" );
            if ( val >= 0.9f )
            {
                Renderer.Morphs.Clear( "blink_left", BlinkSpeed );
                Renderer.Morphs.Clear( "blink_right", BlinkSpeed );
            }
            if ( val <= 0.1f && Renderer.Morphs.Get( "blink_left" ) <= 0.1f )
            {
                isBlinking = false;
                nextBlink = BlinkInterval + Random.Shared.Float( -1.5f, 1.5f );
            }
        }
    }

    void UpdateLipSync()
    {
        // Через CitizenAnimationHelper
        if ( AnimHelper != null )
        {
            AnimHelper.VoiceLevel = VoiceLevel;
        }

        // Дополнительно через морфы для лучшего качества
        Renderer.Morphs.Set( "jaw_open", VoiceLevel * 0.5f, 0.05f );
    }

    void UpdateEmotion()
    {
        // Эмоция задаётся индексом (0 = нейтральная)
        float fadeTime = 0.5f;

        // Сначала сбрасываем
        Renderer.Morphs.Clear( "smile", fadeTime );
        Renderer.Morphs.Clear( "frown", fadeTime );
        Renderer.Morphs.Clear( "brow_down_left", fadeTime );
        Renderer.Morphs.Clear( "brow_down_right", fadeTime );

        switch ( EmotionIndex )
        {
            case 1: // Радость
                Renderer.Morphs.Set( "smile", 0.7f, fadeTime );
                break;
            case 2: // Грусть
                Renderer.Morphs.Set( "frown", 0.6f, fadeTime );
                break;
            case 3: // Злость
                Renderer.Morphs.Set( "brow_down_left", 0.8f, fadeTime );
                Renderer.Morphs.Set( "brow_down_right", 0.8f, fadeTime );
                break;
        }
    }
}
```

---

## API Справочник

### MorphAccessor (через SkinnedModelRenderer.Morphs)

| Метод | Описание |
|-------|----------|
| `Set( string name, float weight )` | Установить значение морфа (0-1) |
| `Set( string name, float weight, float fadeTime )` | Установить с плавным переходом |
| `Get( string name )` | Получить текущее значение морфа |
| `Clear( string name )` | Сбросить значение до 0 |
| `Clear( string name, float fadeTime )` | Сбросить с плавным переходом |
| `ContainsOverride( string name )` | Проверить, задано ли переопределение |
| `Names` | Массив всех доступных морфов |

### ModelMorphs (через Model.Morphs)

| Свойство/Метод | Описание |
|----------------|----------|
| `Count` | Количество морфов в модели |
| `Names` | Массив имён всех морфов |
| `GetName( int index )` | Имя морфа по индексу |
| `GetIndex( string name )` | Индекс морфа по имени |

---

## Следующий шаг

Перейдите к разделу **[Разрушаемые объекты](../15-Breakables-Shatter/README.md)** для изучения системы разрушаемых предметов.
