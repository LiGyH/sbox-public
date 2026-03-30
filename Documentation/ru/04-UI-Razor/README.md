# 4. UI система — Razor

> ⚠️ **Самый важный раздел для многих разработчиков!** Razor — это шаблонизатор для создания UI в s&box. Он может казаться сложным, но на самом деле это просто HTML + C#.

## Что такое Razor?

**Razor** — это технология от Microsoft, которая позволяет смешивать **HTML-разметку** и **код C#** в одном файле. В s&box Razor используется для создания **пользовательских интерфейсов** (UI): HUD, меню, инвентарь, чат и т.д.

### Razor в s&box — это:
- **HTML-подобная разметка** — описываем структуру интерфейса
- **C# код** — логика интерфейса
- **SCSS стили** — внешний вид (описано в следующем разделе)

---

## Основы Razor

### Структура файла .razor

```razor
@* Это комментарий в Razor *@

@* 1. Подключаем пространства имён *@
@using Sandbox;
@using Sandbox.UI;

@* 2. Указываем базовый класс *@
@inherits PanelComponent

@* 3. Пространство имён этого компонента *@
@namespace MyGame.UI

@* 4. Разметка (HTML-подобная) *@
<root>
    <div class="my-panel">
        <h1>Привет мир!</h1>
        <p>Это мой первый UI компонент</p>
    </div>
</root>

@* 5. Код C# *@
@code
{
    // Здесь пишем свойства и методы
    [Property]
    public string Title { get; set; } = "Заголовок";
    
    protected override int BuildHash() => System.HashCode.Combine( Title );
}
```

### Разбор по частям

| Часть | Описание |
|-------|----------|
| `@using Sandbox;` | Подключаем пространства имён (чтобы не писать полные пути) |
| `@inherits PanelComponent` | Наследуемся от PanelComponent — это делает наш файл **компонентом сцены** |
| `@namespace MyGame.UI` | Имя пространства, в котором будет класс |
| `<root>` | Корневой элемент UI. **Всё содержимое должно быть внутри `<root>`** |
| `@code { }` | Блок C# кода — свойства, методы, логика |
| `BuildHash()` | Определяет, когда перерисовывать компонент |

---

## Типы Razor-компонентов

### 1. PanelComponent (компонент сцены)

Используется когда UI привязан к **объекту в сцене** (как обычный компонент):

```razor
@using Sandbox;
@using Sandbox.UI;
@inherits PanelComponent
@namespace MyGame.UI

<root>
    <div class="hud">
        <div class="health">Здоровье: @Health</div>
        <div class="ammo">Патроны: @Ammo</div>
    </div>
</root>

@code
{
    [Property] public float Health { get; set; } = 100f;
    [Property] public int Ammo { get; set; } = 30;
    
    protected override int BuildHash() => System.HashCode.Combine( Health, Ammo );
}
```

**Как использовать:**
1. Создайте файл `Hud.razor`
2. Создайте файл `Hud.razor.scss` (стили)
3. Добавьте компонент `Hud` на объект в сцене (или на объект с `ScreenPanel`)

### 2. Panel (вложенный компонент)

Используется для **переиспользуемых** компонентов UI, которые вставляются внутрь других:

```razor
@using Sandbox.UI;
@inherits Panel
@namespace MyGame.UI

<root>
    <div class="health-bar">
        <div class="fill" style="width: @(Percent)%" />
        <label>@Current / @Max</label>
    </div>
</root>

@code
{
    public float Current { get; set; }
    public float Max { get; set; } = 100f;
    public float Percent => (Current / Max * 100f).Clamp( 0, 100 );
    
    protected override int BuildHash() => System.HashCode.Combine( Current, Max );
}
```

---

## Вывод данных: символ @

Символ `@` — это главный инструмент Razor. Он позволяет вставлять C# выражения прямо в разметку:

```razor
<root>
    @* Простое свойство *@
    <div>@PlayerName</div>
    
    @* Выражение с вычислением *@
    <div>Здоровье: @(Health)%</div>
    
    @* Форматирование *@
    <div>Деньги: @Money.ToString("C")</div>
    <div>Время: @TimeSpan.FromSeconds(GameTime).ToString(@"mm\:ss")</div>
    
    @* Вызов метода *@
    <div>@GetStatusText()</div>
    
    @* Тернарный оператор *@
    <div class="@(IsAlive ? "alive" : "dead")">
        @(IsAlive ? "Жив" : "Мёртв")
    </div>
</root>

@code
{
    [Property] public string PlayerName { get; set; } = "Игрок";
    [Property] public float Health { get; set; } = 100f;
    [Property] public float Money { get; set; } = 1500f;
    [Property] public float GameTime { get; set; }
    [Property] public bool IsAlive { get; set; } = true;
    
    string GetStatusText()
    {
        if ( Health > 75 ) return "Отличное состояние";
        if ( Health > 25 ) return "Ранен";
        return "Критическое состояние!";
    }
    
    protected override int BuildHash() => System.HashCode.Combine(
        PlayerName, Health, Money, GameTime, IsAlive
    );
}
```

---

## Условный рендеринг: @if / @else

```razor
<root>
    @* Простое условие *@
    @if ( IsAlive )
    {
        <div class="alive-panel">
            <div class="health">❤️ @Health</div>
        </div>
    }
    else
    {
        <div class="death-screen">
            <h1>ВЫ ПОГИБЛИ</h1>
            <button onclick="@Respawn">Возродиться</button>
        </div>
    }
    
    @* Множественные условия *@
    @if ( Health > 75 )
    {
        <div class="status good">Отлично</div>
    }
    else if ( Health > 25 )
    {
        <div class="status warning">Осторожно</div>
    }
    else
    {
        <div class="status danger">Критично!</div>
    }
    
    @* Условное отображение элемента *@
    @if ( ShowMinimap )
    {
        <div class="minimap">
            @* Содержимое миникарты *@
        </div>
    }
    
    @* Проверка на null *@
    @if ( CurrentWeapon != null )
    {
        <div class="weapon-info">
            <div class="weapon-name">@CurrentWeapon.Name</div>
            <div class="weapon-ammo">@CurrentWeapon.Ammo</div>
        </div>
    }
</root>

@code
{
    [Property] public bool IsAlive { get; set; } = true;
    [Property] public float Health { get; set; } = 100f;
    [Property] public bool ShowMinimap { get; set; } = true;
    public WeaponInfo CurrentWeapon { get; set; }
    
    void Respawn()
    {
        Health = 100f;
        IsAlive = true;
    }
    
    protected override int BuildHash() => System.HashCode.Combine(
        IsAlive, Health, ShowMinimap, CurrentWeapon?.Name, CurrentWeapon?.Ammo
    );
}
```

---

## Циклы: @foreach / @for

```razor
<root>
    @* === СПИСОК ИГРОКОВ === *@
    <div class="player-list">
        <h2>Игроки онлайн:</h2>
        @foreach ( var player in Players )
        {
            <div class="player-row">
                <span class="name">@player.Name</span>
                <span class="score">@player.Score</span>
                <span class="ping">@(player.Ping)мс</span>
            </div>
        }
    </div>
    
    @* === ИНВЕНТАРЬ (сетка предметов) === *@
    <div class="inventory-grid">
        @for ( int i = 0; i < InventorySize; i++ )
        {
            var item = GetItem( i );
            <div class="slot @(item != null ? "has-item" : "empty")"
                 onclick="@(() => SelectSlot(i))">
                @if ( item != null )
                {
                    <img src="@item.Icon" />
                    <span class="count">@item.Count</span>
                }
                else
                {
                    <span class="slot-number">@(i + 1)</span>
                }
            </div>
        }
    </div>
    
    @* === СПИСОК С ИНДЕКСАМИ === *@
    <div class="quest-log">
        @{ int index = 0; }
        @foreach ( var quest in Quests )
        {
            <div class="quest @(quest.Completed ? "completed" : "")">
                <span class="number">@(++index).</span>
                <span class="title">@quest.Title</span>
                @if ( quest.Completed )
                {
                    <span class="check">✅</span>
                }
            </div>
        }
    </div>
</root>

@code
{
    public List<PlayerInfo> Players { get; set; } = new();
    public int InventorySize { get; set; } = 20;
    public List<QuestInfo> Quests { get; set; } = new();
    
    ItemInfo GetItem( int slot )
    {
        // Логика получения предмета по слоту
        return null;
    }
    
    void SelectSlot( int slot )
    {
        Log.Info( $"Выбран слот: {slot}" );
    }
    
    protected override int BuildHash() => System.HashCode.Combine(
        Players.Count, InventorySize, Quests.Count
    );
    
    // Вспомогательные классы
    public class PlayerInfo
    {
        public string Name { get; set; }
        public int Score { get; set; }
        public int Ping { get; set; }
    }
    
    public class ItemInfo
    {
        public string Icon { get; set; }
        public int Count { get; set; }
    }
    
    public class QuestInfo
    {
        public string Title { get; set; }
        public bool Completed { get; set; }
    }
}
```

---

## События (onclick, onmouseover и т.д.)

```razor
<root>
    @* === ПРОСТОЙ КЛИК === *@
    <button onclick="@OnButtonClick">Нажми меня!</button>
    
    @* === КЛИК С ПАРАМЕТРОМ === *@
    <button onclick="@(() => BuyItem("sword"))">Купить меч</button>
    <button onclick="@(() => BuyItem("shield"))">Купить щит</button>
    
    @* === НАВЕДЕНИЕ МЫШИ === *@
    <div class="item" 
         onmouseover="@(() => ShowTooltip("Меч героя"))"
         onmouseout="@HideTooltip">
        <img src="sword.png" />
    </div>
    
    @* === СОБЫТИЯ МЫШИ === *@
    <div class="draggable"
         onmousedown="@OnDragStart"
         onmouseup="@OnDragEnd">
        Перетащи меня
    </div>
    
    @* === TOOLTIP ПРИ НАВЕДЕНИИ === *@
    @if ( !string.IsNullOrEmpty(TooltipText) )
    {
        <div class="tooltip">@TooltipText</div>
    }
</root>

@code
{
    string TooltipText { get; set; }
    
    void OnButtonClick()
    {
        Log.Info( "Кнопка нажата!" );
    }
    
    void BuyItem( string itemName )
    {
        Log.Info( $"Покупаем: {itemName}" );
    }
    
    void ShowTooltip( string text )
    {
        TooltipText = text;
    }
    
    void HideTooltip()
    {
        TooltipText = null;
    }
    
    void OnDragStart()
    {
        Log.Info( "Начало перетаскивания" );
    }
    
    void OnDragEnd()
    {
        Log.Info( "Конец перетаскивания" );
    }
    
    protected override int BuildHash() => System.HashCode.Combine( TooltipText );
}
```

---

## Вложенные компоненты (RenderFragment / ChildContent)

### Компонент-контейнер (Card.razor)

```razor
@using Sandbox.UI;
@inherits Panel
@namespace MyGame.UI

<root>
    <div class="card-header">
        <h2>@Title</h2>
    </div>
    <div class="card-body">
        @ChildContent
    </div>
</root>

@code
{
    [Parameter] public string Title { get; set; } = "Карточка";
    [Parameter] public RenderFragment ChildContent { get; set; }
    
    protected override int BuildHash() => System.HashCode.Combine( Title );
}
```

### Использование вложенного компонента

```razor
@using Sandbox;
@using Sandbox.UI;
@inherits PanelComponent
@namespace MyGame.UI

<root>
    @* Используем наш компонент Card *@
    <Card Title="Информация об игроке">
        <div class="player-info">
            <div>Имя: @PlayerName</div>
            <div>Уровень: @Level</div>
            <div>Опыт: @Experience</div>
        </div>
    </Card>
    
    <Card Title="Инвентарь">
        <div class="inventory">
            @foreach ( var item in Items )
            {
                <div class="item">@item</div>
            }
        </div>
    </Card>
    
    <Card Title="Статистика">
        <div class="stats">
            <div>Убийств: @Kills</div>
            <div>Смертей: @Deaths</div>
            <div>K/D: @(Deaths > 0 ? (Kills / (float)Deaths).ToString("F2") : "∞")</div>
        </div>
    </Card>
</root>

@code
{
    [Property] public string PlayerName { get; set; } = "Игрок";
    [Property] public int Level { get; set; } = 1;
    [Property] public int Experience { get; set; } = 0;
    [Property] public int Kills { get; set; }
    [Property] public int Deaths { get; set; }
    public List<string> Items { get; set; } = new() { "Меч", "Щит", "Зелье" };
    
    protected override int BuildHash() => System.HashCode.Combine(
        PlayerName, Level, Experience, Kills, Deaths, Items.Count
    );
}
```

### Компонент с именованными слотами

```razor
@* Page.razor — компонент страницы с боковой панелью *@
@using Sandbox.UI;
@inherits Panel
@namespace MyGame.UI

<root>
    <div class="page-layout">
        @if ( Sidebar != null )
        {
            <div class="sidebar">@Sidebar</div>
        }
        <div class="main-content">
            @if ( Header != null )
            {
                <div class="page-header">@Header</div>
            }
            <div class="page-body">@Body</div>
            @if ( Footer != null )
            {
                <div class="page-footer">@Footer</div>
            }
        </div>
    </div>
</root>

@code
{
    [Parameter] public RenderFragment Sidebar { get; set; }
    [Parameter] public RenderFragment Header { get; set; }
    [Parameter] public RenderFragment Body { get; set; }
    [Parameter] public RenderFragment Footer { get; set; }
    
    protected override int BuildHash() => 0;
}
```

### Использование именованных слотов

```razor
<Page>
    <Sidebar>
        <nav>
            <button onclick="@(() => CurrentPage = "home")">Главная</button>
            <button onclick="@(() => CurrentPage = "shop")">Магазин</button>
            <button onclick="@(() => CurrentPage = "stats")">Статистика</button>
        </nav>
    </Sidebar>
    <Header>
        <h1>@GetPageTitle()</h1>
    </Header>
    <Body>
        @if ( CurrentPage == "home" )
        {
            <div>Добро пожаловать!</div>
        }
        else if ( CurrentPage == "shop" )
        {
            <div>Магазин предметов</div>
        }
    </Body>
    <Footer>
        <div class="copyright">© Моя игра 2025</div>
    </Footer>
</Page>
```

---

## Ссылки на элементы: @ref

С помощью `@ref` можно получить программный доступ к HTML-элементу:

```razor
<root>
    <div class="progress-bar">
        <div class="fill" @ref="FillBar" />
    </div>
    
    <input type="text" @ref="SearchInput" placeholder="Поиск..." />
</root>

@code
{
    public Panel FillBar { get; set; }
    public Panel SearchInput { get; set; }
    
    [Property] public float Progress { get; set; } = 0.5f;
    
    protected override void OnAfterTreeRender( bool firstTime )
    {
        if ( FillBar != null )
        {
            // Программно устанавливаем ширину
            FillBar.Style.Width = Length.Percent( Progress * 100f );
        }
        
        if ( firstTime && SearchInput != null )
        {
            // Фокус при первом рендере
            SearchInput.Focus();
        }
    }
    
    protected override int BuildHash() => System.HashCode.Combine( Progress );
}
```

---

## Управление CSS-классами

```razor
<root>
    @* Способ 1: Прямое указание класса *@
    <div class="panel active">Статический класс</div>
    
    @* Способ 2: Динамический класс через выражение *@
    <div class="panel @(IsActive ? "active" : "")">Динамический класс</div>
    
    @* Способ 3: Несколько условных классов *@
    <div class="player-card @GetCardClasses()">
        @PlayerName
    </div>
    
    @* Способ 4: Для каждого элемента списка *@
    @foreach ( var tab in Tabs )
    {
        <div class="tab @(tab == CurrentTab ? "selected" : "")"
             onclick="@(() => CurrentTab = tab)">
            @tab
        </div>
    }
</root>

@code
{
    [Property] public bool IsActive { get; set; }
    [Property] public string PlayerName { get; set; } = "Player";
    [Property] public bool IsAlive { get; set; } = true;
    [Property] public bool IsAdmin { get; set; }
    
    public string CurrentTab { get; set; } = "Главная";
    public List<string> Tabs { get; set; } = new() { "Главная", "Магазин", "Настройки" };
    
    string GetCardClasses()
    {
        var classes = new List<string>();
        if ( IsAlive ) classes.Add( "alive" );
        else classes.Add( "dead" );
        if ( IsAdmin ) classes.Add( "admin" );
        return string.Join( " ", classes );
    }
    
    protected override int BuildHash() => System.HashCode.Combine(
        IsActive, IsAlive, IsAdmin, CurrentTab
    );
}
```

---

## ScreenPanel и WorldPanel

### ScreenPanel — UI на экране

`ScreenPanel` — компонент, который создаёт **корневую панель** для UI на экране (2D):

```
Настройка:
1. Создайте GameObject
2. Добавьте компонент ScreenPanel
3. Добавьте ваш PanelComponent (например, Hud) на ЭТОТ ЖЕ объект
```

```csharp
// Или программно:
public sealed class HudSetup : Component
{
    protected override void OnAwake()
    {
        // ScreenPanel обычно добавляется через редактор
        // Но можно и программно:
        var screenPanel = Components.Get<ScreenPanel>();
        if ( screenPanel == null )
        {
            screenPanel = Components.Create<ScreenPanel>();
        }
    }
}
```

### WorldPanel — UI в мире (3D)

`WorldPanel` рисует UI **в 3D пространстве** (например, имя игрока над головой):

```razor
@* NameTag.razor — имя над головой игрока *@
@using Sandbox;
@using Sandbox.UI;
@inherits PanelComponent
@namespace MyGame.UI

<root>
    <div class="nametag">
        <div class="name">@PlayerName</div>
        <div class="health-bar">
            <div class="fill" style="width: @(HealthPercent)%" />
        </div>
    </div>
</root>

@code
{
    [Property] public string PlayerName { get; set; } = "Игрок";
    [Property] public float Health { get; set; } = 100f;
    [Property] public float MaxHealth { get; set; } = 100f;
    
    float HealthPercent => (Health / MaxHealth * 100f).Clamp( 0, 100 );
    
    protected override int BuildHash() => System.HashCode.Combine( PlayerName, Health );
}
```

Настройка:
1. Создайте дочерний GameObject у игрока
2. Добавьте компонент `WorldPanel`
3. Добавьте `NameTag` компонент
4. Расположите на нужной высоте (LocalPosition = `(0, 0, 80)` например)

---

## Ввод текста

```razor
@using Sandbox;
@using Sandbox.UI;
@inherits PanelComponent
@namespace MyGame.UI

<root>
    <div class="chat">
        @* Сообщения чата *@
        <div class="messages">
            @foreach ( var msg in Messages )
            {
                <div class="message">
                    <span class="author" style="color: @msg.Color">@msg.Author:</span>
                    <span class="text">@msg.Text</span>
                </div>
            }
        </div>
        
        @* Поле ввода *@
        <div class="input-row">
            <TextEntry @ref="ChatInput" 
                       Value="@InputText" 
                       ValueChanged="@OnTextChanged"
                       Placeholder="Введите сообщение..."
                       OnSubmit="@SendMessage" />
            <button onclick="@SendMessage">Отправить</button>
        </div>
    </div>
</root>

@code
{
    public TextEntry ChatInput { get; set; }
    public string InputText { get; set; } = "";
    
    public List<ChatMessage> Messages { get; set; } = new();
    
    void OnTextChanged( string value )
    {
        InputText = value;
    }
    
    void SendMessage()
    {
        if ( string.IsNullOrWhiteSpace( InputText ) ) return;
        
        Messages.Add( new ChatMessage
        {
            Author = "Вы",
            Text = InputText,
            Color = "#00ff00"
        });
        
        InputText = "";
    }
    
    protected override int BuildHash() => System.HashCode.Combine(
        Messages.Count, InputText
    );
    
    public class ChatMessage
    {
        public string Author { get; set; }
        public string Text { get; set; }
        public string Color { get; set; }
    }
}
```

---

## BuildHash — когда перерисовывать?

`BuildHash()` — это критически важный метод. Он определяет, **когда компонент нужно перерисовать**.

### Как это работает:
1. s&box вызывает `BuildHash()` каждый кадр
2. Если хеш **изменился** — компонент **перерисовывается**
3. Если хеш **тот же** — ничего не происходит (экономия ресурсов)

```csharp
// ❌ ПЛОХО: Всегда перерисовывается
protected override int BuildHash() => Random.Shared.Next();

// ❌ ПЛОХО: Никогда не перерисовывается (статический UI)
protected override int BuildHash() => 0;

// ✅ ХОРОШО: Перерисовывается только когда данные изменились
protected override int BuildHash() => System.HashCode.Combine( Health, Ammo, IsAlive );

// ✅ ХОРОШО: Для списков учитываем количество элементов
protected override int BuildHash() => System.HashCode.Combine( 
    Items.Count, 
    SelectedItem?.Name,
    CurrentPage 
);
```

---

## Полный пример: Игровой HUD

```razor
@* GameHud.razor — полноценный игровой HUD *@
@using Sandbox;
@using Sandbox.UI;
@inherits PanelComponent
@namespace MyGame.UI

<root>
    @* Панель здоровья (левый низ) *@
    <div class="health-section">
        <div class="health-icon">❤️</div>
        <div class="health-bar">
            <div class="health-fill" style="width: @(HealthPercent)%" />
            <label class="health-text">@((int)Health) / @((int)MaxHealth)</label>
        </div>
    </div>
    
    @* Оружие (правый низ) *@
    @if ( HasWeapon )
    {
        <div class="weapon-section">
            <div class="weapon-name">@WeaponName</div>
            <div class="ammo">
                <span class="current">@CurrentAmmo</span>
                <span class="separator">/</span>
                <span class="reserve">@ReserveAmmo</span>
            </div>
        </div>
    }
    
    @* Перекрестье (центр) *@
    <div class="crosshair">+</div>
    
    @* Уведомления (правый верх) *@
    <div class="notifications">
        @foreach ( var notification in Notifications )
        {
            <div class="notification @notification.Type">
                @notification.Text
            </div>
        }
    </div>
    
    @* Таблица очков (по нажатию Tab) *@
    @if ( ShowScoreboard )
    {
        <div class="scoreboard">
            <div class="scoreboard-header">
                <span>Игрок</span>
                <span>Убийства</span>
                <span>Смерти</span>
                <span>Пинг</span>
            </div>
            @foreach ( var player in ScoreboardPlayers )
            {
                <div class="scoreboard-row @(player.IsLocal ? "local" : "")">
                    <span>@player.Name</span>
                    <span>@player.Kills</span>
                    <span>@player.Deaths</span>
                    <span>@(player.Ping)ms</span>
                </div>
            }
        </div>
    }
</root>

@code
{
    // Свойства здоровья
    [Property] public float Health { get; set; } = 100f;
    [Property] public float MaxHealth { get; set; } = 100f;
    float HealthPercent => (Health / MaxHealth * 100f).Clamp( 0, 100 );
    
    // Свойства оружия
    [Property] public bool HasWeapon { get; set; } = true;
    [Property] public string WeaponName { get; set; } = "AK-47";
    [Property] public int CurrentAmmo { get; set; } = 30;
    [Property] public int ReserveAmmo { get; set; } = 90;
    
    // Таблица очков
    public bool ShowScoreboard { get; set; }
    public List<ScoreboardEntry> ScoreboardPlayers { get; set; } = new();
    
    // Уведомления
    public List<NotificationInfo> Notifications { get; set; } = new();
    
    protected override void OnUpdate()
    {
        // Показываем таблицу по зажатию Tab
        ShowScoreboard = Input.Down( "Score" );
    }
    
    public void AddNotification( string text, string type = "info" )
    {
        Notifications.Add( new NotificationInfo { Text = text, Type = type } );
        
        // Удаляем через 3 секунды (упрощённо)
        // В реальном проекте используйте таймер
    }
    
    protected override int BuildHash() => System.HashCode.Combine(
        Health, MaxHealth, HasWeapon, WeaponName, CurrentAmmo,
        ReserveAmmo, ShowScoreboard, Notifications.Count
    );
    
    public class ScoreboardEntry
    {
        public string Name { get; set; }
        public int Kills { get; set; }
        public int Deaths { get; set; }
        public int Ping { get; set; }
        public bool IsLocal { get; set; }
    }
    
    public class NotificationInfo
    {
        public string Text { get; set; }
        public string Type { get; set; } = "info";
    }
}
```

---

## Частые ошибки и их решения

### ❌ Ошибка 1: Забыли `<root>`
```razor
@* НЕПРАВИЛЬНО — нет корневого элемента *@
<div>Привет</div>

@* ПРАВИЛЬНО *@
<root>
    <div>Привет</div>
</root>
```

### ❌ Ошибка 2: Стили в отдельном .scss файле (не .razor.scss)
```
❌ MyPanel.scss        — движок НЕ УВИДИТ этот файл!
✅ MyPanel.razor.scss  — правильно! Движок подхватит стили
```

### ❌ Ошибка 3: Не обновляется UI
```csharp
// Забыли добавить переменную в BuildHash!
[Property] public int Score { get; set; }

// ❌ Score не в BuildHash — UI не обновится
protected override int BuildHash() => System.HashCode.Combine( Health );

// ✅ Добавили Score
protected override int BuildHash() => System.HashCode.Combine( Health, Score );
```

### ❌ Ошибка 4: Скобки в выражениях
```razor
@* ❌ Без скобок — ошибка компиляции *@
<div>@Health + 10</div>

@* ✅ С круглыми скобками *@
<div>@(Health + 10)</div>
```

---

## Следующий шаг

Перейдите к разделу **[UI система — SCSS стили](../05-UI-SCSS-Styling/README.md)** для изучения стилизации интерфейса.
