% Время жизни

Эта глава является одной из трёх, описывающих систему владения ресурсами Rust.
Эта система представляет собой наиболее уникальную и привлекательную особенность
Rust, о которой разработчики должны иметь полное представление. Владение — это
то, как Rust достигает своей главной цели — безопасности памяти. Система
владения включает в себя несколько различных концепций, каждая из которых
рассматривается в своей собственной главе:

* [владение][ownership], ключевая концепция
* [заимствование][borrowing], и связанная с ним возможность «ссылки»
* время жизни, её вы читаете сейчас

Эти три главы взаимосвязаны, и их порядок важен. Вы должны будете освоить все
три главы, чтобы полностью понять систему владения.

[ownership]: ownership.html
[borrowing]: references-and-borrowing.html

# Мета

Прежде чем перейти к подробностям, отметим два важных момента в системе
владения.

Rust сфокусирован на безопасности и скорости. Это достигается за счёт
«абстракций с нулевой стоимостью» (zero-cost abstractions). Это значит, что в
Rust стоимость абстракций должна быть настолько малой, насколько это возможно
без ущерба для работоспособности. Система владения ресурсами — это яркий пример
абстракции с нулевой стоимостью. Весь анализ, о котором мы будем говорить в этом
руководстве, выполняется _во время компиляции_. Во время исполнения вы не
платите за какую-либо из возможностей ничего.

Тем не менее, эта система всё же имеет определённую стоимость: кривая обучения.
Многие пользователи Rust занимаются тем, что мы зовём «борьбой с проверкой
заимствования» — компилятор Rust отказывается компилировать программу, которая
по мнению автора является абсолютно правильной. Это часто происходит потому, что
мысленное представление программиста о том, как должно работать владение, не
совпадает с реальными правилами, которыми оперирует Rust. Вы, наверное, поначалу
также будете испытывать подобные трудности. Однако существует и хорошая новость:
более опытные разработчики на Rust говорят, что чем больше они работают с
правилами системы владения, тем меньше они борются с компилятором.

Имея это в виду, давайте перейдём к изучению системы владения.

# Время жизни

Одалживание ссылки на ресурс, которым кто-то владеет, может быть довольно
сложным. Например, представьте себе следующую последовательность операций:

- Мы получаем абстрактную ссылку на какой-то ресурс.
- Мы одалживаем вам ссылку на этот ресурс.
- Мы решаем, что ресурс нам больше не требуется, и освобождаем его, в то время
  как у вас все еще есть на него ссылка.
- Вы решаете использовать этот ресурс.

Ой-ой! Ваша ссылка указывает на недопустимый ресурс. Это называется «висячий
указатель» или «использование после освобождения», когда ресурсом является
память.

Чтобы исправить это, мы должны убедиться, что четвертый шаг никогда не
произойдет после третьего. Система владения в Rust делает это через понятие
времени жизни, которое описывает область видимости, на протяжении которой ссылка
будет действительна.

Когда у нас есть функция, которая принимает ссылку в качестве аргумента, мы
можем явно или неявно указать время жизни ссылки:

```rust
// неявно
fn foo(x: &i32) {
}

// явно
fn bar<'a>(x: &'a i32) {
}
```

Читается `'a` как «время жизни a». Технически, все ссылки имеют некоторое время
жизни, связанное с ними, но компилятор позволяет опускать его в общих случаях.
Прежде чем мы перейдем к этому, давайте разберем пример ниже, с явным указанием
времени жизни:

```rust,ignore
fn bar<'a>(...)
```

Эта часть объявляет параметры времени жизни. Она говорит, что `bar` имеет один
параметр времени жизни, `'a`. Если бы в качестве параметров функции у нас было
две ссылки, то это выглядело бы так:

```rust,ignore
fn bar<'a, 'b>(...)
```

Затем в списке параметров функции мы используем заданные параметры времени
жизни:

```rust,ignore
...(x: &'a i32)
```

Если бы мы хотели `&mut` ссылку, то сделали бы так:

```rust,ignore
...(x: &'a mut i32)
```

Если вы сравните `&mut i32` с `&'a mut i32`, то увидите, что они отличаются
только определением времени жизни `'a`, написанным между `&` и `mut i32`. `&mut
i32` читается как «изменяемая ссылка на i32», а `&'a mut i32` — как «изменяемая
ссылка на i32 со временем жизни 'a».

## Внутри [`struct`][struct]'ов

Вы также должны будете явно указать время жизни при работе со
[`struct`][struct]'ми:

```rust
struct Foo<'a> {
    x: &'a i32,
}

fn main() {
    let y = &5; // то же самое, что и `let _y = 5; let y = &_y;`
    let f = Foo { x: y };

    println!("{}", f.x);
}
```

[struct]: structs.html

Как вы можете заметить, структуры также могут иметь время жизни. Так же как и
функции,

```rust
struct Foo<'a> {
# x: &'a i32,
# }
```

объявляет время жизни и

```rust
# struct Foo<'a> {
x: &'a i32,
# }
```

использует его. Почему же мы должны определять время жизни здесь? Мы должны
убедиться, что ссылка на `Foo` не может жить дольше, чем ссылка на `i32`,
содержащаяся в нем.

## Блоки `impl`

Давайте реализуем метод для `Foo`:

```rust
struct Foo<'a> {
    x: &'a i32,
}

impl<'a> Foo<'a> {
    fn x(&self) -> &'a i32 { self.x }
}

fn main() {
    let y = &5; // то же самое, что и `let _y = 5; let y = &_y;`
    let f = Foo { x: y };

    println!("x is: {}", f.x());
}
```
Как вы можете видеть, нам нужно объявить время жизни для `Foo` в строке с
`impl`. Мы повторяем `'a` дважды, как в функциях: `impl<'a>` определяет  время
жизни `'a`, и `Foo<'a>` использует его.

## Несколько времён жизни (Multiple lifetimes)

Если вы имеете несколько ссылок, вы можете использовать одно и то же время
жизни несколько раз:

```rust
fn x_or_y<'a>(x: &'a str, y: &'a str) -> &'a str {
```

Этот код говорит, что `x` и `y` находятся в одной области видимости друг с
другом, и что возвращаемое значение живо на протяжении той же области видимости.
Если вы хотите, чтобы `x` и `y` имели разные времена жизни, вы должны
использовать параметры нескольких времён жизни:


```rust
fn x_or_y<'a, 'b>(x: &'a str, y: &'b str) -> &'a str {
```

В этом примере `x` и `y` имеют различные области видимости, но возвращаемое
значение имеет то же время жизни, что и `x`.

## Осмысливаем области видимости (Thinking in scopes)

Один из способов понять, что же такое время жизни — это визуализировать область,
в которой ссылка является действительной. Например:

```rust
fn main() {
    let y = &5;     // -+ y входит в область видимости
                    //  |
    // что-то       //  |
                    //  |
}                   // -+ y выходит из области видимости
```

Добавим нашу структуру `Foo`:

```rust
struct Foo<'a> {
    x: &'a i32,
}

fn main() {
    let y = &5;           // -+ y входит в область видимости
    let f = Foo { x: y }; // -+ f входит в область видимости
    // что-то             //  |
                          //  |
}                         // -+ f и y выходят из области видимости
```

Наша `f` живет в области видимости `y`, поэтому все работает. Что же произойдёт,
если это будет не так? Этот код не будет работать:

```rust,ignore
struct Foo<'a> {
    x: &'a i32,
}

fn main() {
    let x;                    // -+ x входит в область видимости
                              //  |
    {                         //  |
        let y = &5;           // ---+ y входит в область видимости
        let f = Foo { x: y }; // ---+ f входит в область видимости
        x = &f.x;             //  | | здесь ошибка
    }                         // ---+ f и y выходят из области видимости
                              //  |
    println!("{}", x);        //  |
}                             // -+ x выходит из области видимости
```

Уф! Как вы можете видеть здесь, области видимости `f` и `y` меньше, чем область
видимости `x`. Но когда мы выполняем `x = &f.x`, мы присваиваем `x` ссылку на
что-то, что вот-вот выйдет из области видимости.

Присвоение имени времени жизни — это способ задать имя области видимости. Чтобы
думать о чём-то, нужно иметь название для этого.

## 'static

Время жизни с именем «static» — особенное. Оно обозначает, что что-то имеет
время жизни, равное времени жизни всей программы. Большинство программистов на
Rust впервые сталкиваются с `'static`, когда имеют дело со строками:

```rust
let x: &'static str = "Привет, мир.";
```

Строковые литералы имеют тип `&'static str`, потому что ссылка всегда
действительна: строки располагаются в сегменте данных конечного двоичного файла.
Другой пример — глобальные переменные:

```rust
static FOO: i32 = 5;
let x: &'static i32 = &FOO;
```

В этом примере `i32` добавляется в сегмент данных двоичного файла, а `x`
ссылается на него.

## Опускание времени жизни

В Rust есть мощный локальный вывод типов. Однако, сигнатуры объявлений верхнего
уровня не выводятся, чтобы можно было рассуждать о типах на основании одних лишь
сигнатур. Из соображений удобства, введён ограниченный механизм вывода типов
сигнатур функций, называемый «опускание времени жизни» («lifetime elision»). Он
выводит типы на основании только элементов сигнатуры — тело функции при этом не
учитывается. При этом его назначение — это вывести лишь параметры времени жизни
аргументов. Для этого он реализует три простых правила. Таким образом, опускание
времени жизни упрощает написание сигнатур, одновременно не скрывая реальные типы
аргументов.

Когда речь идет о неявном времени жизни, мы используем термины *входное время
жизни* (*input lifetime*) и *выходное время жизни* (*output lifetime*). *Входное
время жизни* связано с передаваемыми в функцию параметрами, а *выходное время
жизни* связано с возвращаемым функцией значением. Например, эта функция имеет
входное время жизни:

```rust,ignore
fn foo<'a>(bar: &'a str)
```

А эта имеет выходное время жизни:

```rust,ignore
fn foo<'a>() -> &'a str
```

Эта же имеет как входное, так и выходное время жизни:

```rust,ignore
fn foo<'a>(bar: &'a str) -> &'a str
```

Ниже представлены три правила:

* Каждое неявное время жизни в аргументах функции становится отдельным
  временем жизни.

* Если есть ровно одно входное время жизни, явное или неявное, то это время
  жизни назначается всем неявным выходным временам жизни.

* Если есть несколько входных времён жизни, но одно из них это `&self` или `&mut
  self`, то всем неявным выходным временам жизни назначается время жизни `self`.

В противном случае, неявное задание выходного времени жизни является ошибкой.

### Примеры

Вот некоторые примеры функций, представленные в двух видах: с явно и неявно
заданным временем жизни:

```rust,ignore
fn print(s: &str); // неявно
fn print<'a>(s: &'a str); // явно

fn debug(lvl: u32, s: &str); // неявно
fn debug<'a>(lvl: u32, s: &'a str); // явно

// В предыдущем примере для `lvl` не требуется указывать время жизни, потому что
// это не ссылка (`&`). Только элементы, связанные с ссылками (например, такие
// как структура, содержащая ссылку) требуют указания времени жизни.

fn substr(s: &str, until: u32) -> &str; // неявно
fn substr<'a>(s: &'a str, until: u32) -> &'a str; // явно

fn get_str() -> &str; // НЕКОРРЕКТНО, нет входных параметров

fn frob(s: &str, t: &str) -> &str; // НЕКОРРЕКТНО, два входных параметра
fn frob<'a, 'b>(s: &'a str, t: &'b str) -> &str; // Развёрнуто: Выходное время жизни неясно

fn get_mut(&mut self) -> &mut T; // неявно
fn get_mut<'a>(&'a mut self) -> &'a mut T; // явно

fn args<T:ToCStr>(&mut self, args: &[T]) -> &mut Command // неявно
fn args<'a, 'b, T:ToCStr>(&'a mut self, args: &'b [T]) -> &'a mut Command // явно

fn new(buf: &mut [u8]) -> BufWriter; // неявно
fn new<'a>(buf: &'a mut [u8]) -> BufWriter<'a> // явно
```
