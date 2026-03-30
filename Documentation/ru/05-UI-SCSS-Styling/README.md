# 5. UI система — SCSS стили

> ⚠️ **КРИТИЧЕСКИ ВАЖНО:** Файлы стилей должны иметь расширение **`.razor.scss`**, а НЕ просто `.scss`! Движок s&box видит только файлы `МоёИмя.razor.scss`, привязанные к `МоёИмя.razor`.

## Что такое SCSS?

**SCSS** (Sassy CSS) — это расширенная версия CSS с дополнительными возможностями:
- **Вложенность** — стили внутри стилей
- **Переменные** — `$color-primary: #3273eb;`
- **Миксины** — переиспользуемые блоки стилей
- **Математика** — вычисления прямо в стилях

В s&box SCSS используется для стилизации Razor-компонентов.

---

## Как работают стили в s&box

### Правило именования файлов

```
МойКомпонент.razor       ← Razor шаблон (разметка + логика)
МойКомпонент.razor.scss  ← Стили для этого компонента (ОБЯЗАТЕЛЬНО .razor.scss!)
```

**Примеры:**
```
✅ Hud.razor + Hud.razor.scss           — ПРАВИЛЬНО
✅ PlayerCard.razor + PlayerCard.razor.scss — ПРАВИЛЬНО
❌ Hud.razor + Hud.scss                 — НЕ РАБОТАЕТ! Движок не видит .scss
❌ Hud.razor + styles.css               — НЕ РАБОТАЕТ!
```

### Как движок загружает стили

Когда вы создаёте `PanelComponent`, движок:
1. Находит файл `.razor`
2. Ищет файл с таким же именем, но с `.razor.scss` на конце
3. Автоматически загружает стили

---

## Основы SCSS в s&box

### Базовая структура

```scss
// МойКомпонент.razor.scss

// Корневой селектор — имя вашего компонента (или root)
root {
    // Позиционирование — занимаем весь экран
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    
    // Flexbox — главный способ расположения элементов
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
}
```

### Вложенность (Nesting)

```scss
// Вместо длинных селекторов:
// .panel { }
// .panel .header { }
// .panel .header .title { }

// В SCSS можно вкладывать:
.panel {
    background-color: #1a1a2e;
    border-radius: 8px;
    padding: 16px;
    
    .header {
        display: flex;
        justify-content: space-between;
        margin-bottom: 12px;
        
        .title {
            font-size: 20px;
            font-weight: bold;
            color: white;
        }
        
        .close-btn {
            cursor: pointer;
            color: #888;
            
            &:hover {
                color: white; // & ссылается на родителя (.close-btn)
            }
        }
    }
    
    .body {
        padding: 8px;
        color: #ccc;
    }
}
```

### Символ & (амперсанд)

`&` — ссылка на **текущий селектор**. Используется для:

```scss
.button {
    background-color: #3273eb;
    padding: 8px 16px;
    color: white;
    cursor: pointer;
    transition: all 150ms ease;
    
    // &:hover = .button:hover
    &:hover {
        background-color: #4a8af5;
        transform: translateY(-2px);
    }
    
    // &:active = .button:active (при нажатии)
    &:active {
        transform: translateY(0);
        background-color: #2860d0;
    }
    
    // &.primary = .button.primary (кнопка с дополнительным классом)
    &.primary {
        background-color: #31EB31;
    }
    
    // &.danger = .button.danger
    &.danger {
        background-color: #eb3131;
    }
    
    // &.disabled = .button.disabled
    &.disabled {
        opacity: 0.5;
        pointer-events: none;
    }
}
```

---

## Переменные SCSS

```scss
// Объявление переменных
$bg-primary: #1a1a2e;
$bg-secondary: #16213e;
$bg-card: #0f3460;

$text-primary: #ffffff;
$text-secondary: #a0a0b0;
$text-muted: #666680;

$accent-blue: #3273eb;
$accent-green: #31eb31;
$accent-red: #eb3131;
$accent-yellow: #ebc931;

$font-main: "Inter";
$font-heading: "Poppins";

$radius-sm: 4px;
$radius-md: 8px;
$radius-lg: 12px;
$radius-xl: 16px;

$spacing-xs: 4px;
$spacing-sm: 8px;
$spacing-md: 16px;
$spacing-lg: 24px;
$spacing-xl: 32px;

// Использование
.card {
    background-color: $bg-card;
    border-radius: $radius-md;
    padding: $spacing-md;
    font-family: $font-main;
    
    .title {
        color: $text-primary;
        font-family: $font-heading;
        font-size: 20px;
    }
    
    .subtitle {
        color: $text-secondary;
        font-size: 14px;
    }
}
```

---

## Все CSS-свойства s&box

### Позиционирование

```scss
.element {
    // Position
    position: absolute;        // absolute | relative
    top: 10px;
    left: 20px;
    right: 0;
    bottom: 0;
    z-index: 10;              // Порядок наложения (выше = поверх)
}
```

### Размеры

```scss
.element {
    width: 200px;              // Ширина в пикселях
    width: 50%;                // Ширина в процентах от родителя
    height: 100px;
    
    min-width: 100px;          // Минимальная ширина
    max-width: 500px;          // Максимальная ширина
    min-height: 50px;
    max-height: 300px;
    
    aspect-ratio: 16 / 9;     // Соотношение сторон
}
```

### Flexbox (главный инструмент расположения!)

```scss
// Flexbox — основной способ расположения элементов в s&box UI
.container {
    display: flex;             // Включаем Flexbox
    
    // Направление элементов
    flex-direction: row;       // row (горизонтально) | column (вертикально)
    
    // Выравнивание по ГЛАВНОЙ оси
    justify-content: center;       // center | flex-start | flex-end | space-between | space-around
    
    // Выравнивание по ПОБОЧНОЙ оси
    align-items: center;           // center | flex-start | flex-end | stretch
    
    // Перенос строк
    flex-wrap: wrap;               // wrap | nowrap
    
    // Расстояние между элементами
    gap: 10px;
    row-gap: 10px;
    column-gap: 20px;
}

// Flex-свойства дочерних элементов
.child {
    flex-grow: 1;              // Занимать свободное место (0 = нет, 1+ = да)
    flex-shrink: 0;            // Сжиматься при нехватке места (0 = нет)
    flex-basis: 200px;         // Начальный размер
    align-self: flex-end;      // Своё выравнивание
    order: 2;                  // Порядок (меньше = раньше)
}
```

### Примеры Flexbox

```scss
// Горизонтальное меню
.menu {
    display: flex;
    flex-direction: row;
    gap: 8px;
    align-items: center;
    
    .menu-item {
        padding: 8px 16px;
    }
}

// Вертикальный список
.list {
    display: flex;
    flex-direction: column;
    gap: 4px;
    
    .list-item {
        padding: 12px;
    }
}

// Центрирование по обеим осям
.centered {
    display: flex;
    justify-content: center;
    align-items: center;
    width: 100%;
    height: 100%;
}

// Шапка сайта (лого слева, меню справа)
.header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 0 20px;
}

// Сетка карточек
.card-grid {
    display: flex;
    flex-wrap: wrap;
    gap: 16px;
    
    .card {
        flex-basis: 200px;
        flex-grow: 1;
        max-width: 300px;
    }
}
```

### Отступы (Padding / Margin)

```scss
.element {
    // Внутренние отступы (padding)
    padding: 16px;                       // Все стороны
    padding: 8px 16px;                   // Вертикальные горизонтальные
    padding: 8px 16px 12px 16px;         // Верх право низ лево
    padding-top: 10px;
    padding-right: 20px;
    padding-bottom: 10px;
    padding-left: 20px;
    
    // Внешние отступы (margin)
    margin: 10px;
    margin-top: 20px;
    margin-bottom: auto;    // Полезно для прижатия к низу
}
```

### Рамки (Border)

```scss
.element {
    // Толщина рамки
    border: 2px solid #3273eb;
    
    // По отдельности
    border-left: 3px solid red;
    border-top: 1px solid rgba(255, 255, 255, 0.1);
    
    // Скругление углов
    border-radius: 8px;                      // Все углы
    border-top-left-radius: 12px;            // Отдельный угол
    border-top-right-radius: 12px;
    border-bottom-left-radius: 0;
    border-bottom-right-radius: 0;
    
    // Круглый элемент
    border-radius: 50%;
}
```

### Текст и шрифты

```scss
.text {
    // Шрифт
    font-family: "Poppins";          // Название шрифта
    font-size: 16px;                 // Размер
    font-weight: bold;               // normal | bold | 100-900
    font-style: italic;              // normal | italic
    
    // Цвет текста
    color: white;
    color: #3273eb;
    color: rgba(255, 255, 255, 0.8);
    
    // Выравнивание
    text-align: center;              // left | center | right
    
    // Декорация
    text-decoration: underline;
    text-decoration: line-through;
    
    // Трансформация
    text-transform: uppercase;       // uppercase | lowercase | none
    
    // Межстрочный интервал
    line-height: 1.5;
    
    // Межбуквенный интервал
    letter-spacing: 2px;
    
    // Перенос текста
    white-space: nowrap;             // Не переносить
    text-overflow: ellipsis;         // Обрезать с троеточием "..."
    overflow: hidden;                // Скрыть выходящий текст
    word-break: break-all;           // Переносить по буквам
}
```

### Фон (Background)

```scss
.element {
    // Сплошной цвет
    background-color: #1a1a2e;
    background-color: rgba(0, 0, 0, 0.8);    // С прозрачностью
    
    // Градиент
    background-color: linear-gradient(to bottom, #1a1a2e, #0f3460);
    background-color: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    
    // Изображение
    background-image: url("ui/background.png");
    background-size: cover;           // cover | contain | 100% 100%
    background-position: center;
}
```

### Прозрачность и видимость

```scss
.element {
    opacity: 0.8;                     // 0 = невидим, 1 = видим
    pointer-events: none;             // Элемент не реагирует на мышь
    pointer-events: all;              // Элемент реагирует на мышь
    cursor: pointer;                  // Курсор «рука» при наведении
}
```

### Тени

```scss
.element {
    // Тень блока
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
    box-shadow: 0 0 20px rgba(50, 115, 235, 0.5); // Свечение
    
    // Тень текста
    text-shadow: 0 2px 4px rgba(0, 0, 0, 0.5);
}
```

### Переполнение (Overflow)

```scss
.element {
    overflow: hidden;                 // Скрыть выходящий контент
    overflow: scroll;                 // Прокрутка
    overflow-x: hidden;              // Только по горизонтали
    overflow-y: scroll;              // Только по вертикали
}
```

### Фильтры и эффекты

```scss
.element {
    // Размытие фона (стеклянный эффект)
    backdrop-filter-blur: 20px;
    backdrop-filter-brightness: 0.8;
    backdrop-filter-saturate: 1.5;
    
    // Фильтры элемента
    filter-blur: 5px;                 // Размытие
    filter-brightness: 1.2;           // Яркость
    filter-contrast: 1.1;             // Контраст
    filter-saturate: 0;               // Насыщенность (0 = ч/б)
    filter-hue-rotate: 90deg;         // Сдвиг оттенка
    filter-invert: 1;                 // Инверсия
}
```

---

## Псевдоклассы

Псевдоклассы — это состояния элемента:

```scss
.button {
    background-color: #3273eb;
    transition: all 150ms ease;
    
    // При наведении мыши
    &:hover {
        background-color: #4a8af5;
        transform: scale(1.05);
    }
    
    // При нажатии
    &:active {
        background-color: #2860d0;
        transform: scale(0.95);
    }
    
    // При фокусе (Tab-навигация)
    &:focus {
        outline: 2px solid #3273eb;
        outline-offset: 2px;
    }
}
```

### Специальные псевдоклассы s&box

s&box добавляет уникальные псевдоклассы:

```scss
.panel {
    // :intro — анимация ПОЯВЛЕНИЯ элемента
    &:intro {
        opacity: 0;
        transform: translateY(20px);
    }
    
    // :outro — анимация ИСЧЕЗНОВЕНИЯ элемента
    &:outro {
        opacity: 0;
        transform: translateY(-20px);
    }
    
    // Если задать transition, переход будет плавным:
    transition: all 300ms ease;
}

// Пример: плавное появление модального окна
.modal {
    opacity: 1;
    transform: scale(1);
    transition: all 200ms ease;
    
    &:intro {
        opacity: 0;
        transform: scale(0.8);
    }
    
    &:outro {
        opacity: 0;
        transform: scale(0.8);
    }
}

// Пример: выезжающая панель
.slide-panel {
    transform: translateX(0);
    transition: transform 300ms ease;
    
    &:intro {
        transform: translateX(-100%);
    }
    
    &:outro {
        transform: translateX(100%);
    }
}
```

---

## Анимации (@keyframes)

```scss
// Определяем анимацию
@keyframes pulse {
    0% {
        transform: scale(1);
        opacity: 1;
    }
    50% {
        transform: scale(1.1);
        opacity: 0.8;
    }
    100% {
        transform: scale(1);
        opacity: 1;
    }
}

@keyframes spin {
    0% {
        transform: rotate(0deg);
    }
    100% {
        transform: rotate(360deg);
    }
}

@keyframes slideIn {
    0% {
        transform: translateX(-100%);
        opacity: 0;
    }
    100% {
        transform: translateX(0);
        opacity: 1;
    }
}

@keyframes fadeIn {
    from { opacity: 0; }
    to { opacity: 1; }
}

// Применяем анимацию
.pulsing-element {
    animation-name: pulse;
    animation-duration: 1s;
    animation-iteration-count: infinite;   // infinite = бесконечно
    animation-timing-function: ease;       // ease | linear | ease-in | ease-out
}

.spinner {
    animation-name: spin;
    animation-duration: 0.5s;
    animation-iteration-count: infinite;
}

.notification {
    animation-name: slideIn;
    animation-duration: 300ms;
    animation-iteration-count: 1;          // Один раз
}
```

---

## Переходы (Transitions)

Transitions — плавное изменение свойств:

```scss
.button {
    background-color: #3273eb;
    transform: scale(1);
    opacity: 1;
    
    // Плавный переход для ВСЕХ свойств
    transition: all 200ms ease;
    
    &:hover {
        background-color: #4a8af5;
        transform: scale(1.05);
    }
    
    &:active {
        transform: scale(0.95);
    }
}

// Разные скорости для разных свойств
.card {
    background-color: #1a1a2e;
    transform: translateY(0);
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.2);
    
    // Можно указать несколько переходов через запятую
    transition: 
        background-color 200ms ease,
        transform 300ms ease,
        box-shadow 400ms ease;
    
    &:hover {
        background-color: #2a2a3e;
        transform: translateY(-4px);
        box-shadow: 0 8px 24px rgba(0, 0, 0, 0.4);
    }
}
```

---

## Transform (трансформации)

```scss
.element {
    // Перемещение
    transform: translateX(50px);
    transform: translateY(-20px);
    transform: translate(50px, -20px);    // Если поддерживается
    
    // Масштабирование
    transform: scale(1.5);                // Увеличить в 1.5 раза
    transform: scale(0.5);                // Уменьшить в 2 раза
    transform: scaleX(2);                 // Растянуть по горизонтали
    
    // Вращение
    transform: rotate(45deg);
    transform: rotate(-90deg);
    
    // Комбинация
    transform: translateY(-10px) scale(1.1) rotate(5deg);
    
    // Точка трансформации
    transform-origin-x: 50%;              // Центр по X
    transform-origin-y: 0%;               // Верх по Y
}
```

---

## Звуковые эффекты в CSS

Уникальная фича s&box — звуки при наведении/клике прямо в CSS!

```scss
.button {
    // Звук при наведении
    sound-in: "ui.button.over";
    
    // Звук при уходе мыши
    sound-out: "ui.button.press";
    
    &:hover {
        background-color: lighten($bg-primary, 10%);
    }
}
```

---

## Импорт стилей (@import)

Вы можете создать файл с общими переменными и миксинами:

### _variables.scss (файл с переменными)

```scss
// code/UI/styles/_variables.scss

// Цвета
$primary: #3273eb;
$secondary: #31eb31;
$danger: #eb3131;
$warning: #ebc931;

$bg-dark: #0a0a1a;
$bg-medium: #1a1a2e;
$bg-light: #2a2a3e;

$text-white: #ffffff;
$text-gray: #a0a0b0;

// Размеры
$radius: 8px;
$spacing: 16px;
$font-size-sm: 12px;
$font-size-md: 14px;
$font-size-lg: 20px;
$font-size-xl: 28px;
```

### _mixins.scss (файл с миксинами)

```scss
// code/UI/styles/_mixins.scss

// Миксин для карточек
@mixin card {
    background-color: $bg-medium;
    border-radius: $radius;
    padding: $spacing;
    border: 1px solid rgba(255, 255, 255, 0.1);
}

// Миксин для кнопок
@mixin button($color: $primary) {
    background-color: $color;
    padding: 8px 16px;
    border-radius: $radius;
    color: white;
    cursor: pointer;
    font-weight: bold;
    transition: all 150ms ease;
    
    &:hover {
        filter-brightness: 1.2;
        transform: translateY(-1px);
    }
    
    &:active {
        filter-brightness: 0.9;
        transform: translateY(0);
    }
}

// Миксин для свечения
@mixin glow($color: $primary) {
    box-shadow: 0 0 10px rgba($color, 0.3), 0 0 20px rgba($color, 0.1);
}

// Миксин для текста с обрезкой
@mixin text-truncate {
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
}
```

### Использование в компоненте

```scss
// MyPanel.razor.scss
@import "styles/_variables.scss";
@import "styles/_mixins.scss";

root {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    
    .sidebar {
        @include card;
        width: 300px;
        
        .title {
            font-size: $font-size-lg;
            color: $text-white;
            @include text-truncate;
        }
    }
    
    .action-button {
        @include button($primary);
    }
    
    .danger-button {
        @include button($danger);
    }
    
    .selected-item {
        @include glow($secondary);
    }
}
```

---

## Полные примеры стилей

### Пример 1: HUD шутера

```scss
// GameHud.razor.scss

$hud-bg: rgba(0, 0, 0, 0.6);
$health-color: #eb3131;
$armor-color: #3273eb;
$ammo-color: #ebc931;

root {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    pointer-events: none; // Не блокировать мышь
    font-family: "Poppins";
    
    // ==================
    // Полоска здоровья (левый низ)
    // ==================
    .health-section {
        position: absolute;
        left: 20px;
        bottom: 20px;
        display: flex;
        align-items: center;
        gap: 12px;
        
        .health-icon {
            font-size: 32px;
        }
        
        .health-bar {
            width: 200px;
            height: 24px;
            background-color: $hud-bg;
            border-radius: 12px;
            overflow: hidden;
            position: relative;
            
            .health-fill {
                height: 100%;
                background-color: $health-color;
                border-radius: 12px;
                transition: width 300ms ease;
            }
            
            .health-text {
                position: absolute;
                top: 0;
                left: 0;
                right: 0;
                bottom: 0;
                display: flex;
                justify-content: center;
                align-items: center;
                color: white;
                font-size: 12px;
                font-weight: bold;
                text-shadow: 0 1px 2px rgba(0, 0, 0, 0.8);
            }
        }
    }
    
    // ==================
    // Оружие и патроны (правый низ)
    // ==================
    .weapon-section {
        position: absolute;
        right: 20px;
        bottom: 20px;
        text-align: right;
        
        .weapon-name {
            color: white;
            font-size: 14px;
            opacity: 0.7;
            margin-bottom: 4px;
        }
        
        .ammo {
            display: flex;
            align-items: baseline;
            gap: 4px;
            justify-content: flex-end;
            
            .current {
                font-size: 48px;
                font-weight: bold;
                color: $ammo-color;
            }
            
            .separator {
                font-size: 24px;
                color: rgba(255, 255, 255, 0.3);
            }
            
            .reserve {
                font-size: 20px;
                color: rgba(255, 255, 255, 0.5);
            }
        }
    }
    
    // ==================
    // Перекрестье (центр)
    // ==================
    .crosshair {
        position: absolute;
        top: 50%;
        left: 50%;
        transform: translate(-50%, -50%);
        color: white;
        font-size: 24px;
        text-shadow: 0 0 4px rgba(0, 0, 0, 0.8);
        opacity: 0.8;
    }
    
    // ==================
    // Уведомления (правый верх)
    // ==================
    .notifications {
        position: absolute;
        top: 20px;
        right: 20px;
        display: flex;
        flex-direction: column;
        gap: 8px;
        max-width: 300px;
        
        .notification {
            padding: 10px 16px;
            border-radius: 8px;
            color: white;
            font-size: 13px;
            backdrop-filter-blur: 10px;
            transition: all 300ms ease;
            
            &:intro {
                opacity: 0;
                transform: translateX(50px);
            }
            
            &:outro {
                opacity: 0;
                transform: translateX(50px);
            }
            
            &.info {
                background-color: rgba(50, 115, 235, 0.8);
            }
            
            &.success {
                background-color: rgba(49, 235, 49, 0.8);
            }
            
            &.danger {
                background-color: rgba(235, 49, 49, 0.8);
            }
            
            &.warning {
                background-color: rgba(235, 201, 49, 0.8);
                color: #1a1a2e;
            }
        }
    }
    
    // ==================
    // Таблица очков
    // ==================
    .scoreboard {
        position: absolute;
        top: 50%;
        left: 50%;
        transform: translate(-50%, -50%);
        background-color: rgba(0, 0, 0, 0.9);
        border-radius: 12px;
        padding: 20px;
        min-width: 500px;
        backdrop-filter-blur: 20px;
        pointer-events: all;
        
        &:intro {
            opacity: 0;
            transform: translate(-50%, -50%) scale(0.9);
        }
        
        .scoreboard-header {
            display: flex;
            padding: 8px 12px;
            color: rgba(255, 255, 255, 0.5);
            font-size: 12px;
            text-transform: uppercase;
            border-bottom: 1px solid rgba(255, 255, 255, 0.1);
            
            span {
                flex-grow: 1;
                
                &:first-child {
                    flex-grow: 3;
                }
            }
        }
        
        .scoreboard-row {
            display: flex;
            padding: 8px 12px;
            color: white;
            font-size: 14px;
            border-radius: 4px;
            transition: background-color 150ms ease;
            
            span {
                flex-grow: 1;
                
                &:first-child {
                    flex-grow: 3;
                }
            }
            
            &:hover {
                background-color: rgba(255, 255, 255, 0.05);
            }
            
            &.local {
                background-color: rgba(50, 115, 235, 0.2);
                
                &:hover {
                    background-color: rgba(50, 115, 235, 0.3);
                }
            }
        }
    }
}
```

### Пример 2: Стили инвентаря

```scss
// Inventory.razor.scss

$slot-size: 64px;
$slot-gap: 4px;
$slot-bg: rgba(255, 255, 255, 0.05);
$slot-hover: rgba(255, 255, 255, 0.1);
$slot-selected: rgba(50, 115, 235, 0.3);
$slot-border: rgba(255, 255, 255, 0.1);
$rarity-common: #9d9d9d;
$rarity-uncommon: #1eff00;
$rarity-rare: #0070dd;
$rarity-epic: #a335ee;
$rarity-legendary: #ff8000;

root {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    pointer-events: all;
    
    .inventory-window {
        background-color: rgba(10, 10, 26, 0.95);
        border: 1px solid rgba(255, 255, 255, 0.1);
        border-radius: 12px;
        padding: 20px;
        backdrop-filter-blur: 20px;
        
        &:intro {
            opacity: 0;
            transform: scale(0.9);
        }
        
        transition: all 200ms ease;
        
        .title {
            font-size: 20px;
            font-weight: bold;
            color: white;
            margin-bottom: 16px;
            padding-bottom: 8px;
            border-bottom: 1px solid rgba(255, 255, 255, 0.1);
        }
        
        .slots {
            display: flex;
            flex-wrap: wrap;
            gap: $slot-gap;
            width: calc(($slot-size + $slot-gap) * 5); // 5 слотов в ряд
            
            .slot {
                width: $slot-size;
                height: $slot-size;
                background-color: $slot-bg;
                border: 1px solid $slot-border;
                border-radius: 6px;
                position: relative;
                cursor: pointer;
                transition: all 150ms ease;
                display: flex;
                justify-content: center;
                align-items: center;
                
                &:hover {
                    background-color: $slot-hover;
                    border-color: rgba(255, 255, 255, 0.3);
                    transform: scale(1.05);
                }
                
                &.selected {
                    background-color: $slot-selected;
                    border-color: rgba(50, 115, 235, 0.5);
                    box-shadow: 0 0 10px rgba(50, 115, 235, 0.3);
                }
                
                &.has-item {
                    .item-icon {
                        width: 80%;
                        height: 80%;
                    }
                    
                    .item-count {
                        position: absolute;
                        bottom: 2px;
                        right: 4px;
                        font-size: 11px;
                        font-weight: bold;
                        color: white;
                        text-shadow: 0 1px 2px black;
                    }
                }
                
                &.empty {
                    .slot-number {
                        color: rgba(255, 255, 255, 0.15);
                        font-size: 11px;
                    }
                }
                
                // Рамки по редкости
                &.common { border-color: $rarity-common; }
                &.uncommon { border-color: $rarity-uncommon; }
                &.rare { border-color: $rarity-rare; }
                &.epic { border-color: $rarity-epic; }
                &.legendary { 
                    border-color: $rarity-legendary;
                    box-shadow: 0 0 8px rgba($rarity-legendary, 0.3);
                }
            }
        }
    }
}
```

### Пример 3: Главное меню

```scss
// MainMenu.razor.scss

root {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background-color: linear-gradient(135deg, #0a0a1a 0%, #1a1a3e 100%);
    display: flex;
    justify-content: center;
    align-items: center;
    font-family: "Poppins";
    
    &:intro {
        opacity: 0;
    }
    
    transition: opacity 500ms ease;
    
    .menu-container {
        display: flex;
        flex-direction: column;
        align-items: center;
        gap: 32px;
        
        .logo {
            font-size: 64px;
            font-weight: bold;
            color: white;
            text-shadow: 0 0 20px rgba(50, 115, 235, 0.5);
            letter-spacing: 4px;
            
            @keyframes logo-glow {
                0% { text-shadow: 0 0 20px rgba(50, 115, 235, 0.3); }
                50% { text-shadow: 0 0 40px rgba(50, 115, 235, 0.6); }
                100% { text-shadow: 0 0 20px rgba(50, 115, 235, 0.3); }
            }
            
            animation-name: logo-glow;
            animation-duration: 3s;
            animation-iteration-count: infinite;
        }
        
        .menu-buttons {
            display: flex;
            flex-direction: column;
            gap: 12px;
            min-width: 300px;
            
            .menu-button {
                padding: 16px 32px;
                background-color: rgba(255, 255, 255, 0.05);
                border: 1px solid rgba(255, 255, 255, 0.1);
                border-radius: 8px;
                color: white;
                font-size: 18px;
                text-align: center;
                cursor: pointer;
                transition: all 200ms ease;
                
                &:hover {
                    background-color: rgba(50, 115, 235, 0.3);
                    border-color: rgba(50, 115, 235, 0.5);
                    transform: translateX(10px);
                }
                
                &:active {
                    transform: translateX(5px);
                    background-color: rgba(50, 115, 235, 0.5);
                }
                
                // Появление кнопок с задержкой
                &:intro {
                    opacity: 0;
                    transform: translateX(-30px);
                }
            }
        }
        
        .version {
            color: rgba(255, 255, 255, 0.3);
            font-size: 12px;
        }
    }
}
```

---

## Частые ошибки со стилями

### ❌ Ошибка 1: Файл .scss вместо .razor.scss
```
❌ MyPanel.scss         — движок НЕ УВИДИТ
✅ MyPanel.razor.scss   — правильно
```

### ❌ Ошибка 2: Нет `root` селектора
```scss
// ❌ Стили не привязаны к корневому элементу
.my-class { color: red; }

// ✅ Оборачиваем в root
root {
    .my-class { color: red; }
}
```

### ❌ Ошибка 3: Забыли pointer-events
```scss
// ❌ HUD блокирует мышь
root {
    position: absolute;
    top: 0; left: 0; right: 0; bottom: 0;
    // Мышь застревает в UI!
}

// ✅ Отключаем перехват мыши на HUD, включаем на кнопках
root {
    position: absolute;
    top: 0; left: 0; right: 0; bottom: 0;
    pointer-events: none; // HUD не ловит мышь
    
    .button {
        pointer-events: all; // Кнопки ловят мышь
    }
}
```

### ❌ Ошибка 4: Transition не работает
```scss
// ❌ Transition в :hover — не работает
.element {
    background-color: red;
    
    &:hover {
        background-color: blue;
        transition: all 200ms ease; // Поздно!
    }
}

// ✅ Transition в базовом состоянии
.element {
    background-color: red;
    transition: all 200ms ease; // Тут!
    
    &:hover {
        background-color: blue;
    }
}
```

---

## Следующий шаг

Перейдите к разделу **[Сетевая игра (Networking)](../06-Networking/README.md)** для изучения мультиплеера.
