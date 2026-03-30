# 1. Начало работы с s&box

## Что такое s&box?

**s&box** (произносится «сэндбокс») — это игровой движок от Facepunch Studios, созданный как преемник Garry's Mod. Он позволяет создавать многопользовательские игры на языке **C#** с минимальными усилиями.

### Главные особенности:
- **C#** — современный, простой в изучении язык программирования
- **Компоненты** — вся логика строится из переиспользуемых блоков
- **Мультиплеер из коробки** — сетевой код уже встроен в движок
- **Razor UI** — интерфейс создаётся с помощью HTML-подобных шаблонов
- **Горячая перезагрузка** — код обновляется без перезапуска игры

---

## Структура проекта

Когда вы создаёте новый проект в s&box, он имеет следующую структуру:

```
МойПроект/
├── code/                    # Папка с кодом
│   ├── Assembly.cs          # Файл с глобальными using-директивами
│   └── MyComponent.cs       # Ваши компоненты
├── Assets/                  # Модели, текстуры, звуки
├── ProjectSettings/         # Настройки проекта
└── .addon                   # Конфигурация аддона
```

### Файл Assembly.cs

Этот файл содержит глобальные директивы `using`, которые доступны во всех файлах проекта:

```csharp
// Assembly.cs - этот файл автоматически подключает пространства имён для всего проекта
global using System;
global using System.Collections.Generic;
global using System.Linq;
global using System.Threading.Tasks;
global using Sandbox;
```

> 💡 **Что такое `using`?** — Это инструкция, которая говорит компилятору: «Я хочу использовать классы из этого пространства имён без указания полного пути». Например, вместо `Sandbox.Component` можно писать просто `Component`.

---

## Ваш первый компонент

**Компонент** — это основной строительный блок в s&box. Каждый компонент — это класс C#, который наследуется от `Component`.

### Шаг 1: Создайте файл

Создайте файл `code/MyFirstComponent.cs`:

```csharp
// MyFirstComponent.cs
// Это ваш первый компонент! Он вращает объект, на который прикреплён.

public sealed class MyFirstComponent : Component
{
    // [Property] делает это поле видимым в редакторе
    // Вы сможете менять скорость прямо в редакторе, не трогая код!
    [Property]
    public float RotationSpeed { get; set; } = 50.0f;

    // OnUpdate вызывается КАЖДЫЙ КАДР (60+ раз в секунду)
    protected override void OnUpdate()
    {
        // Вращаем объект вокруг оси Y
        // Time.Delta — время между кадрами (обычно ~0.016 секунд)
        // Это нужно, чтобы вращение было одинаковым на любом FPS
        WorldRotation *= Rotation.FromAxis( Vector3.Up, RotationSpeed * Time.Delta );
    }
}
```

### Шаг 2: Добавьте компонент к объекту

1. Создайте или выберите объект в сцене
2. В инспекторе нажмите **"Add Component"**
3. Найдите `MyFirstComponent` и добавьте его
4. Измените `Rotation Speed` если хотите

### Шаг 3: Запустите игру

Нажмите Play — ваш объект начнёт вращаться!

---

## Разбор кода по частям

### Что такое `sealed class`?

```csharp
public sealed class MyFirstComponent : Component
```

- `public` — этот класс доступен отовсюду
- `sealed` — от этого класса нельзя наследоваться (рекомендуется для производительности)
- `class` — это класс (шаблон для создания объектов)
- `: Component` — наследуемся от базового класса `Component`

### Что такое `[Property]`?

```csharp
[Property]
public float RotationSpeed { get; set; } = 50.0f;
```

Атрибут `[Property]` делает свойство **видимым в редакторе** s&box. Без него свойство существует только в коде.

| Атрибут | Описание | Пример |
|---------|----------|--------|
| `[Property]` | Показывает в редакторе | `[Property] public float Speed { get; set; }` |
| `[Property, Range(0, 100)]` | Ползунок от 0 до 100 | `[Property, Range(0, 100)] public float Health { get; set; }` |
| `[Property, TextArea]` | Многострочное текстовое поле | `[Property, TextArea] public string Description { get; set; }` |
| `[Property, Title("Имя")]` | Своё название в редакторе | `[Property, Title("Скорость бега")] public float RunSpeed { get; set; }` |
| `[Property, Category("Движение")]` | Группировка свойств | `[Property, Category("Movement")] public float Speed { get; set; }` |
| `[Property, Hide]` | Скрыть в редакторе | `[Property, Hide] public int InternalValue { get; set; }` |

### Что такое `protected override void OnUpdate()`?

```csharp
protected override void OnUpdate()
{
    // Код, выполняемый каждый кадр
}
```

- `protected` — доступен только внутри класса и его наследников
- `override` — переопределяем метод из базового класса `Component`
- `void` — метод ничего не возвращает
- `OnUpdate()` — вызывается движком каждый кадр

---

## Жизненный цикл компонента

Каждый компонент проходит через определённые этапы. Движок вызывает специальные методы на каждом этапе:

```
Создание объекта
       │
       ▼
  OnAwake()          ← Вызывается один раз при создании
       │
       ▼
  OnEnabled()        ← Вызывается когда компонент включается
       │
       ▼
  ┌─── OnUpdate()    ← Вызывается КАЖДЫЙ КАДР (основной цикл)
  │    OnFixedUpdate() ← Вызывается с фиксированной частотой (физика)
  │         │
  └─────────┘
       │
       ▼
  OnDisabled()       ← Вызывается когда компонент выключается
       │
       ▼
  OnDestroy()        ← Вызывается при удалении объекта
```

### Пример с использованием всех методов жизненного цикла:

```csharp
public sealed class LifecycleExample : Component
{
    [Property]
    public float MaxHealth { get; set; } = 100f;
    
    private float currentHealth;
    private bool isAlive;
    
    // Вызывается ОДИН РАЗ при создании компонента
    protected override void OnAwake()
    {
        Log.Info( "OnAwake: Компонент создан!" );
        currentHealth = MaxHealth;
        isAlive = true;
    }
    
    // Вызывается когда компонент ВКЛЮЧАЕТСЯ
    protected override void OnEnabled()
    {
        Log.Info( "OnEnabled: Компонент включён!" );
    }
    
    // Вызывается КАЖДЫЙ КАДР
    protected override void OnUpdate()
    {
        if ( !isAlive ) return;
        
        // Пример: уменьшаем здоровье со временем
        currentHealth -= 1f * Time.Delta;
        
        if ( currentHealth <= 0 )
        {
            isAlive = false;
            Log.Info( "Объект погиб!" );
            GameObject.Destroy(); // Удаляем объект
        }
    }
    
    // Вызывается с ФИКСИРОВАННОЙ частотой (для физики)
    protected override void OnFixedUpdate()
    {
        // Здесь лучше делать физические расчёты
        // Вызывается ровно 60 раз в секунду
    }
    
    // Вызывается когда компонент ВЫКЛЮЧАЕТСЯ
    protected override void OnDisabled()
    {
        Log.Info( "OnDisabled: Компонент выключен!" );
    }
    
    // Вызывается при УДАЛЕНИИ объекта
    protected override void OnDestroy()
    {
        Log.Info( "OnDestroy: Компонент уничтожен! Очищаем ресурсы." );
    }
}
```

---

## Типы данных, которые вам пригодятся

### Основные типы C#

| Тип | Описание | Пример |
|-----|----------|--------|
| `int` | Целое число | `int score = 0;` |
| `float` | Дробное число | `float speed = 5.5f;` |
| `bool` | Истина/ложь | `bool isAlive = true;` |
| `string` | Текст | `string name = "Игрок";` |
| `double` | Точное дробное число | `double precise = 3.14159;` |

### Типы s&box

| Тип | Описание | Пример |
|-----|----------|--------|
| `Vector3` | 3D-координата (x, y, z) | `Vector3 pos = new Vector3( 0, 0, 100 );` |
| `Vector2` | 2D-координата (x, y) | `Vector2 uv = new Vector2( 0.5f, 0.5f );` |
| `Rotation` | Вращение (кватернион) | `Rotation rot = Rotation.Identity;` |
| `Angles` | Углы Эйлера (pitch, yaw, roll) | `Angles ang = new Angles( 0, 90, 0 );` |
| `Color` | Цвет (r, g, b, a) | `Color color = Color.Red;` |
| `Transform` | Позиция + вращение + масштаб | `Transform t = Transform.Zero;` |
| `BBox` | Ограничивающий прямоугольник | `BBox box = BBox.FromPositionAndSize( pos, 10 );` |
| `Ray` | Луч (начало + направление) | `Ray ray = new Ray( pos, Vector3.Forward );` |

### Примеры работы с Vector3:

```csharp
// Создание
Vector3 position = new Vector3( 100, 200, 50 );
Vector3 zero = Vector3.Zero;          // (0, 0, 0)
Vector3 forward = Vector3.Forward;    // (1, 0, 0) — вперёд
Vector3 up = Vector3.Up;              // (0, 0, 1) — вверх
Vector3 right = Vector3.Right;        // (0, -1, 0) — вправо

// Математика
Vector3 sum = position + Vector3.Up * 100;     // Поднять на 100 единиц вверх
float distance = position.Distance( zero );     // Расстояние между точками
Vector3 direction = (zero - position).Normal;    // Направление от position к zero
float length = position.Length;                  // Длина вектора

// Интерполяция (плавное перемещение)
Vector3 start = new Vector3( 0, 0, 0 );
Vector3 end = new Vector3( 100, 0, 0 );
Vector3 middle = Vector3.Lerp( start, end, 0.5f ); // (50, 0, 0) — середина
```

### Примеры работы с Rotation:

```csharp
// Создание
Rotation identity = Rotation.Identity;          // Без вращения
Rotation fromAngles = Rotation.FromYaw( 90 );   // Повернуть на 90° по горизонтали
Rotation fromAxis = Rotation.FromAxis( Vector3.Up, 45 ); // 45° вокруг оси вверх

// Получение направлений
Vector3 forward = rotation.Forward;  // Куда «смотрит» вращение
Vector3 up = rotation.Up;
Vector3 right = rotation.Right;

// Конвертация в углы
Angles angles = rotation.Angles();
float yaw = angles.yaw;     // Поворот по горизонтали
float pitch = angles.pitch;  // Наклон вверх/вниз
float roll = angles.roll;    // Наклон в сторону

// Плавный поворот
Rotation target = Rotation.LookAt( targetPosition - myPosition );
WorldRotation = Rotation.Slerp( WorldRotation, target, Time.Delta * 5f );
```

---

## Полезные советы для новичков

### 1. Используйте `Log.Info()` для отладки

```csharp
Log.Info( "Позиция игрока: " + WorldPosition );
Log.Info( $"Здоровье: {currentHealth}/{MaxHealth}" );
Log.Warning( "Внимание: здоровье ниже 20!" );
Log.Error( "Ошибка: объект не найден!" );
```

### 2. Доступ к другим компонентам

```csharp
// Получить компонент на ЭТОМ же объекте
var renderer = Components.Get<ModelRenderer>();

// Получить компонент на ДОЧЕРНИХ объектах
var childRenderer = Components.GetInChildren<ModelRenderer>();

// Получить компонент на РОДИТЕЛЬСКИХ объектах
var parentComponent = Components.GetInParent<MyComponent>();

// Получить ВСЕ компоненты определённого типа
var allRenderers = Components.GetAll<ModelRenderer>();
```

### 3. Работа с объектами в сцене

```csharp
// Найти объект по имени
var player = Scene.GetAllObjects( true )
    .FirstOrDefault( x => x.Name == "Player" );

// Создать новый объект
var newObject = new GameObject( true, "МойОбъект" );
newObject.WorldPosition = new Vector3( 0, 0, 100 );

// Добавить компонент к объекту
var renderer = newObject.Components.Create<ModelRenderer>();
renderer.Model = Model.Load( "models/dev/box.vmdl" );

// Удалить объект
someObject.Destroy();
```

### 4. Время и таймеры

```csharp
// Time.Delta — время между кадрами (используйте для плавного движения!)
WorldPosition += Vector3.Forward * Speed * Time.Delta;

// Time.Now — текущее время с начала игры
float gameTime = Time.Now;

// Простой таймер
private TimeUntil nextShot;

protected override void OnUpdate()
{
    if ( nextShot <= 0 )
    {
        Shoot();
        nextShot = 0.5f; // Следующий выстрел через 0.5 секунды
    }
}

// Таймер обратного отсчёта
private TimeSince lastDamage;

protected override void OnUpdate()
{
    if ( lastDamage > 3.0f )
    {
        // Прошло больше 3 секунд с последнего урона
        // Начинаем регенерацию
        Health += 10f * Time.Delta;
    }
}
```

---

## Следующий шаг

Теперь, когда вы понимаете основы, перейдите к разделу **[GameObject и Компоненты](../02-GameObject-Components/README.md)** для более глубокого погружения в систему объектов и компонентов.
