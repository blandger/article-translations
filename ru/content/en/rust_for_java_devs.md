# Rust для Java разработчиков

[Оригинал](https://leshow.github.io/post/rust_for_java_devs/)

Я хотел бы внести изменения в блог и оставить более острые темы, чтобы сосредоточиться на них. Пожалуй, одной из самых важных тем, которые можно сделать в сообществе Rust - это обучение новых разработчиков. Я думал о том, как лучше всего подойти к преподаванию Rust для тех, кто привык работать с Java и помочь освоить язык для нового проекта.

Java был языком, который я изучал в университете, поэтому мой опыт работы с ним несколько анахроничен и я не делал никаких реальных попыток успевать за языком. Когда я в последний раз писал Java, если вы хотели передать функцию в качестве аргумента, вы должны были объявить новый интерфейс или обернуть функцию в `Callable<T>`. C тех пор Java проделал большой путь. Это добавленные функции, которые имеют явное влияние из функционального программирования и происходят из ML. Я говорю о лямбдах, `Optional` типах и т.д. Эта статья не расскажет про то как писать все на Rust или о том, что вам нужно выбросить весь свой Java-код. Java - отличный язык с действительными вариантами использования. Я хочу сделать некоторые сравнения между Java и Rust для начинающего Rust программиста.

## Мотивация языка

Сначала о целях каждого языка. Java был создан, чтобы решить множество проблем. Он должен был быть ["простым, объектно-ориентированным и модульным"](https://en.wikipedia.org/wiki/Java_(programming_language)#Principles), он должен был работать на виртуальной машине, быть переносимым на разные архитектуры и он должен был иметь высокую производительность. Цели Rust - это быть ["невероятно быстрыми и эффективными в отношении памяти"](https://www.rust-lang.org/), иметь "богатую систему типов, безопасность памяти, потоко-безопасность", генерировать хорошие сообщения об ошибках и иметь встроенный менеджер пакетов.

Между ними есть некоторые отличия. Производительность в Rust является высокой, упоминается отсутствие среды времени выполнения, а также безопасность памяти вместе с мощной системой типов. Это области, в которых Rust действительно сияет, его код является производительным и безопасным. Серверы, которые должны обрабатывать многие тысячи запросов в секунду, приложения, которые должны быть быстрыми и работать с небольшим объемом памяти, ОС (операционная система) или код, работающий на встроенном (embedded) устройстве. Эти вещи могут быть выполнены на других языках, но это область для которой был создан Rust. Rust это позволяет, в то же время предотвращает такие вещи как переполнение буфера, некорректные (dangling) или нулевые указатели.

Это не значит, что Rust не используется в других областях (я смотрю на wasm frontends библиотеку [yew](https://github.com/yewstack/yew) ). И лично я обнаружил, что как только я привык к языку, моя скорость создания прототипов часто была на одном уровне или лучше, чем с языками, которые имеют более простую (или динамическую) систему типов. Но это аргумент для другого поста в блоге.

С педагогической точки зрения, я думаю, что основные моменты переобучения с Java на Rust сводятся к нескольким широким категориям:

- Алгебраические типы данных
- Отличия в модели памяти (отсутствие сборщика мусора GC)
- Владение
- Времена жизни и заимствование
- Параметрический полиморфизм (обобщения) и типажи

Мне интересно также услышать мнение других людей; если у вас есть мысли о педагогике, не стесняйтесь, пишите мне по электронной почте. Я не буду освещать все это здесь, но мы затронем некоторые важные части.

## Алгебраические типы данных

### Перечисление

Программирование на Rust является подходом гораздо более ориентированным на данные и типы. В Rust есть два основных способа объявления новых типов значений с помощью ключевых слов `enum` и `struct`. `enum` - это тип перечисления (теговое объединение, если предпочитаете так). Оно отличается от `enum` в Java, но если мы собираемся провести сравнение то, думайте о нем как о Java `enum`, которое способно выразить гораздо больше. Общая идея `enum` на любом языке состоит в том, что оно выражает тип, который может иметь разные варианты. Например, вы возможно слышали, что Rust не имеет `null` или `nil`. Это верно и если вы хотите выразить отсутствие значения в Rust, то в стандартной библиотеке [есть для этого тип](https://doc.rust-lang.org/std/option/enum.Option.html).

```rust
enum Option<T> {
    None,
    Some(T),
}
```

Этот код объявляет новый тип `Option`, который принимает параметр типа `T`. Тип `T` неограничен, что означает данное определение действительно для любого типа и мы представляем его с помощью переменной `T`. Тип `Option` имеет 2 возможных варианта, это либо конструктор `None` ("конструктор данных") представляющий отсутствие значения, либо конструктор `Some` со значением `T`. Для объявления значения пишем код:

```rust
let a = Some("foo".to_string()); // объявление значения типа Option<String>

let b: Option<usize> = Some(1); // объявление значения типа Option<usize>, но указывая аннотацию типа
```

Примечание об аннотациях типа. Оно требуется при указании ограничений функций, но локально типы выводятся автоматически. Считается идиоматичным не указывать аннотацию, если не требуется. В некоторых случаях это необходимо, но это выходит за рамки сегодняшнего поста.

Rust включает способ сопоставления с образцом вариантов `enum` с помощью ключевого слова `match`. Если вы раньше не использовали язык с хорошим сопоставлением шаблона, то его использование действительно приятно.

```rust
fn plus(a: Option<usize>) -> Option<usize> {
    match a {
        Some(v) => Some(v + 1),
        None => None
   }
}
```

Это простой случай, но язык шаблонов гораздо более мощный. Можно написать целую статью про `match`. Проверьте список допустимых вариантов синтаксиса на странице [cheats.rs](https://cheats.rs/#pattern-matching) для оператора `match`. Во многих случаях он заменяет выражения `if/else`.

В примере мы берем значение `Option`, добавляем что-то к нему (если это вариант `Some` ) и возвращаем данное значение. Это общий шаблон, но у `Option` есть функции в стандартной библиотеке, чтобы написать это более кратко.

```rust
fn plus(a: Option<usize>) -> Option<usize> {
    a.map(|v| v + 1)
}
```

Пример `|| {}` - это синтаксис замыкания. Замыкания в Rust не требуют какого-либо выделения памяти в куче, что соответствует целям языка, направленного на предоставление "системы с расширенными типами" без потери производительности.

### Структура

Ключевое слово `struct` - это способ объявить новые записи. Это, вероятно, будет более знакомо любому, кто имеет опыт работы с языками основанными на Си.

```rust
struct A {
   field: usize
}
```

Вы  можете объявить "анонимную" запись с помощью синтаксиса кортежа.

```rust
let a: (usize, usize) = (1, 1); // аннотация типа не нужна
```

Структуры могут и часто содержат обобщенные типы.

```rust
struct Foo<T> {
    field: T
}
```

### Блок реализации

Для обоих объявленных типов данных `enum` и `struct` можно написать реализацию. Я считаю, что лучше всего рассматривать `impl` как набор преобразований доступных для вашего типа. Это примерно такое же близкое к понятию ОО, которое мы собираемся получить в Rust. У вас может быть `struct` со значением `impl` и если вы хорошенько прищуритесь, он будет выглядеть *почти* как объект:

```rust
struct Thangs {
   list: Vec<Thang>
}

struct Thang;

impl Thangs {
   // *примечание* функция `new` не имеет специального значения, она больше 'по соглашению'
   fn new() -> Self {
       Self {
           list: vec![]
       }
   }

   fn add_thang(&mut self, thang: Thang) {
       self.list.push(thang);
   }
}

fn main() {
  // *примечание* &mut self в методе 'add_thangs' требует объявления mut в коде ниже
   let mut thangs = Thangs::new();
   thangs.add_thang(Thang);
}
```

В отличие от Java, синтаксис `value.method()` фактически является просто сахаром для "универсального синтаксиса вызова функции". Мы могли бы вызвать метод, передавая `&mut self` в самого себя:

```rust
let mut thangs = Thangs::new();
Thangs::add_thang(&mut thangs, Thang);
```

На мой взгляд, хороший подход к Rust начинается с понимания основных определений типов данных. Типы `enum` и `struct` будут вашим хлебом с маслом. В Java что-то вроде `struct` и `impl` находятся в классах объектов, где ваши данные и методы "сожительствуют", что связывает код (я полагаю, что намеренно). Прежде чем подумать "ну почему Rust не может просто добавить объекты", знайте что Java также получает функцию, похожую на структуру. В приходящей Java 14 "Records" (записи) будут добавлены в язык. Так что на самом деле может случиться так, что следующее поколение кода Java будет выглядеть более "ржавым", чем обратное (я ни в коем случае не утверждаю, что Rust был первым, кто сделал sum и product типы). Я даже видел предложения в Java, которые имеют что-то вроде суммируемых типов, так что давайте, принимайте алгебраические типы данных!

## Стек против Кучи

### Стек память

В Rust нет сборщика мусора. У вас гораздо больше контроля над тем, как вы выделяете память и где живут ваши значения. По умолчанию все существует в стеке. Вот так:

```rust
struct Thing { field: usize }

fn main() {
    let a = Thing { field: 1 };
    // .. do stuff with a
}
```

Код не требует какого-либо динамического выделения памяти. Компилятор может определить точное количество байт, которые он будет занимать в памяти, для этого даже есть типаж `Sized`. Компилятор также может выяснить, как долго это значение является действительным до его удаления. У него есть определенная начальная точка, когда значение было создано и когда оно выходит из области видимости (в конце main) и тогда значение может быть уничтожено.

Сравните это с Java, где вы создаете экземпляры объекта с помощью `new`, что вызывает выделение в куче и неявная ссылка сохраняется в вашей переменной, которая передается по значению.

### Куча

Мы также можем создавать значения в куче с помощью типа `Box` (в stdlib есть и другие умные указатели, которые также выполняют выделение памяти в куче `Rc`, `Arc` и т.д.). Вы можете спросить, почему мы хотели бы выделять память в куче, если мы можем просто поместить все в стек? Ответ может быть различным, но один ответ связан с тем фактом, что не все типы имеют статически доступный размер, поэтому мы можем обратиться к выделению в куче, чтобы размер стал известен (мы сделаем пример этого). В других случаях у вас может быть большой объем данных, например, большая структура, которая при ее перемещении вызовет большое копирование данных, но размещая ее в "оберточный" тип `Box` или другой умный указатель, мы можем уменьшить объем перемещаемых данных за счет данного выделения.

```rust
fn main () {
    let a = Box::new(Thing { field: 1 });
}
```

Значения, выделенные в куче имеют определенные начало и конец жизни. Существует начальная и конечная область видимости, когда данные будут созданы и уничтожены.  Я считаю, что полезно думать о том, что значение переменной `a` на самом деле *выглядит* как будто оно находится в переменной `a`. В первом примере это была структура с одним полем типа `usize` внутри, но здесь значение в стеке на самом деле является указателем на некоторую память в куче.

Другое использование для `Box` это динамическая диспетчеризация. С помощью этого мы можем получить что-то вроде объектов Java,

```rust
trait Foo {}

struct Thing;

impl Foo for Thing {}

struct OtherThing;

impl Foo for OtherThing {}

fn main () {
    let a: Vec<Box<dyn Foo>> = vec![Box::new(OtherThing), Box::new(Thing)]; // указание типа опционально
    // внутри 'вектора' есть два разных экземпляра структуры, но мы как бы
    // удаляем точный тип и теперь есть только значения известные как типаж Foo
    }
```

Не волнуйтесь насчет ключевого слова `trait`, мы также скоро выясним это. Здесь важно понять, что это поведение динамическое и оно не бесплатно. Мы платим тем, что объявляем это явным образом программе и с фактической стоимостью выделения в куче. В сообществе Rust есть споры, но я думаю, что было бы справедливо сказать, что идиоматичным является избегать выделения в куче, если этого легко избежать.

Я обнаружил, что визуальные диаграммы чрезвычайно полезны для того, чтобы погрузиться в это. [Шпаргалка по контейнерам в Rust](https://docs.google.com/presentation/d/1q-c7UAyrUlM-eZyTo1pd8SZ0qwA_wYxmPZVOQkoDmH4/edit#slide=id.p) - отличный ресурс.

Давайте посмотрим на определение другого типа.

```rust
enum List<T> {
    Nil,
    Cons((T, List<T>))
}
```

Это может выглядеть довольно странно.Тип  `List<T>` может иметь значение `Nil` означающее, что мы достигли конца списка или значение кортежа `Cons`, содержащее значение и остальную часть списка. Подумайте, как это будет выглядеть в памяти, если вы создадите для него значение?

А?

Это не работает

```rust
    error[E0072]: recursive type `List` has infinite size
     --> src/lib.rs:7:1
      |
    7 | enum List<T> {
      | ^^^^^^^^^^^^ recursive type has infinite size
    8 |     Nil,
    9 |     Cons((T, List<T>))
      |          ------------ recursive without indirection
      |
      = help: insert indirection (e.g., a `Box`, `Rc`, or `&`) at some point to make `List` representable
```

Сообщения ошибок компилятора находятся на высоком уровне. Показывается причина и часто `help` имеет предлагаемое исправление кода. Это говорит нам о том, что мы не можем создать такой рекурсивный тип без добавления некоторой косвенности, чтобы компилятор мог определить точный размер. Помните, если по умолчанию все находится в стеке, а значения стека должны иметь статически известный размер, то как мы можем иметь связанный список N-размера? Без ссылки или указателя на следующий элемент в списке, как компилятор будет статически определять, сколько памяти использовать? Убедите себя в том, что это правда. Иногда в этом помогает визуальное мышление.

Мы можем это `исправить`, добавив косвенность.

```rust
enum List<T> {
    Nil,
    Cons((T, Box<List<T>>))
}
```

Теперь у нас есть самый ужасный связанный список в мире. Если вам понравился этот пример, попробуйте прочитать статью [слишком много связанных списков](http://cglab.ca/~abeinges/blah/too-many-lists/book/). Время от времени вы можете найти объявления типов умных указателей.

## Типажи

### Мотивирующий пример

Чтобы проиллюстрировать некоторые различия в проблемах кодирования в Java и в Rust, давайте рассмотрим еще одну (по общему признанию игрушечную) проблему:

```rust
enum Shape {
    Circle { radius: f32 },
    Rectangle { width: f32, height: f32 },
}
```

Мы хотим получить значение `area` (площадь) у этого типа, поэтому, возможно, мы сделаем функцию:

```rust
impl Shape {
    pub fn area(self) -> f32 {
        match self {
            Shape::Circle { radius } => std::f32::consts::PI * radius.powi(2),
            Shape::Rectangle { width, height } => width * height,
        }
    }
 }
```

Инкапсуляция и видимость в Java имеют много форм на уровне класса. В Rust функции и типы являются либо `pub` (для публичного пользования), либо нет (для локального внутри модуля). Видимость для других участников контролируется модульной системой, я рекомендую прочитать [Rust reference](https://doc.rust-lang.org/reference/visibility-and-privacy.html).

Вернемся к примеру. В Java мы можем создать родительский класс или интерфейс `Shape` и иметь классы `Circle` и `Rectangle`, каждый из которых реализует метод `area`. Если мы подумаем о различиях между реализацией на Rust и в Java, станет ясно несколько вещей:

- Если мы хотим другуой `Shape`:
    - Java:  нужно просто объявить другой класс и реализовать интерфейс `Shape`
    - Rust: мы должны изменить исходное определение `Shape` *и* везде, где оно используется ( код с `match` не будет компилироваться, если НЕ обрабатывать все возможные варианты)
- Если мы хотим новую функцию в `Shape`:
    - Java: нужно изменить исходный "контракт" `Shape`, то есть мы должны добавить новую функцию в интерфейс и каждый кто реализует этот интерфейс, должен быть изменен
    - Rust: мы можем просто добавить новый `impl`

459/5000<br>Это было описано ранее как 'проблема выразительности'. Это иллюстрирует некоторые основные различия между подходами в языках. Это не умаляет использования  `enum` в Rust или использования интерфейсов/классов в Java. Как правило мы *хотим*, чтобы Rust выполнил исчерпывающий анализ с помощью {`match` и вариантами `enum`. Но возникает вопрос: «Можем ли мы написать это так, чтобы не требовалось модифицировать существующий код?»

Я думаю, что типажи предлагают довольно приятное решение.

```rust
trait Area {
    fn area(self) -> f32; // здесь можно было бы вернуть обобщенный или ассоциированный тип (вместо i32)
}

struct Rectangle {
    width: f32,
    height: f32
}

struct Circle {
    radius: f32
}

impl Area for Rectangle {
    fn area(self) -> f32 {
        self.width * self.height
    }
}

impl Area for Circle {
    fn area(self) -> f32 {
        std::f32::consts::PI * self.radius.powi(2)
    }
}
```

Теперь, если нам нужно добавить новую функцию в `Circle` или в `Rectangle`, скажем `perimeter`, мы можем сделать это без изменения оригинального типажа или типов:

```rust
trait Perimeter {
    fn perimeter(self) -> f32;
}

impl Perimeter for Rectangle {
    fn perimeter(self) -> f32 {
        2. * (self.width + self.height)
    }
}
// и т.д.
```

Мы также можем добавить больше типов фигур. И мы можем написать функции, которые принимают любой тип, который имеет типаж "периметр" или типажи "периметр *и* площадь".

```rust
fn do_something<T: Perimeter + Area>(shape: T) { // принимать только тип, имеющий обе реализации Perimeter и Area
        unimplemented!() // подсказка: данный макрос бесценный
        // он позволяет удовлетворить проверки типа без предоставления реализации
    }
```

### Типажи и обобщения

Реализация типажей в Rust является выразительной. Возможно, вы слышали сравнение ее с перегрузкой операторов, которой нет в Java. Я думаю, что это будет хорошим введением в данный набор функциональности. В Java перегрузку избегают, а в Rust это не так. Типажи предоставлены для вашей реализации и соответствия  спецификации типажа, добиваясь доступа ко встроенному синтаксису и функциональной совместимости. Учтите, что вы можете подключить синтаксис языка с типажами, так работает вся экосистема. Есть типаж `Future` для ожидающих вычислений, есть типажи `Iterator` и `IntoIterator` для использования в `for..in`, есть `Index` для `[]`, не говоря уже о типажах `Add`, `Sub`, `Mul` и другие для арифметических операций. Как минимальный пример, давайте заставим тип работать с типажом <a data-md-type="link" href="https://doc.rust-lang.org/std/ops/trait.Add.html">`Add`</a>

Вот определение `Add` из библиотеки std.

```rust
pub trait Add<Rhs = Self> { // 1
    type Output; // 2
    fn add(self, rhs: Rhs) -> Self::Output; // 3
}
```

Std объявляет типаж `Add` с параметром типа `Rhs`, который по умолчанию равен `Self`, т.е. типу реализающему типаж (1). Он имеет «ассоциированный тип» с именем `Output` (2) и определяет метод `add`, который принимает `self` по значению (становится владельцем `self`) и параметр `rhs` типа `Rhs` (переданный параметр типа) и возвращает тип, ассоциированный с `Output` (3).

```rust
use std::ops::Add;

#[derive(Debug)]
struct Content<T> { // 1
    val: T,
}

impl<T> Add for Content<T> // 2
   where
    T: Add, // 3
{
    type Output = Content<<T as Add>::Output>; // 4
    fn add(self, rhs: Content<T>) -> Self::Output {
        Content {
            val: self.val + rhs.val,
        }
    }
}

fn main() {
    let a = Content { val: 2 };
    let b = Content { val: 5 };
    println!("{:?}", a + b); // Content { val: 7 }
}
```

Мы объявляем новый тип `Content`, который подходит для любого обобщенного `T` (1). В реализации `Add` мы говорим, что `Content` имеет реализацию типажа `Add` (2), при условии, что в типе `Content` *также* есть реализация `Add` (3). После этого мы указываем, что ассоциированный тип `Output` будет типа `Content` типа `Output` от `T`, когда он реализует типаж `Add` (4). Не беспокойтесь, если поначалу все не имеет смысла, как только вы напишите несколько реализаций, понимание придет. Я думаю, это круто, что большая часть этой программы на самом деле о системе типов. У нас есть только несколько строк, которые на самом деле "выполняют работу" и это должно дать вам представление о том, на что похоже программирование в Rust. Вы в основном проектируете на уровне типов, а затем убеждаете Rust компилятор, что ваша программа хорошо сформирована и это позволяет выполнять довольно хорошее парное программирование (с компилятором), указывающим на ваши ошибки.

Следует отметить, что по умолчанию все общие параметры неявно ограничены типажом `Sized` и это означает, что если вы пишете `fn foo<T>(t: T) -> T`, то автоматически подразумевается `T: Sized`. Вы можете отказаться от этого ограничения с помощью `T: ?Sized`, указав что `T` *может* быть не размерным. Для получения дополнительной информации ознакомьтесь с разделом [Динамические типы](https://doc.rust-lang.org/book/ch19-04-advanced-types.html#dynamically-sized-types-and-the-sized-trait) .

## Полиморфизм

В Java полиморфизм обычно означает наследование. Это не совсем так вне мира OO и конечно не так в Rust, в котором вообще отсутствует версия похожая на Java. Полиморфизм Java - это полиморфизм подтипа, тогда как в Rust - это параметрический и специальный (ad-hoc) полиморфизм. Параметрический полиморфизм просто означает, что мы можем передавать параметры обобщенного типа, а специальный ссылается на способ, которым мы можем ограничить эти параметры типа определенными типажами (с помощью конструкции `<T: Trait>` ).

Практически, знание "правильной" терминологии не супер важно. Важно знать, что вы можете объявлять структуры данных с помощью `enum` или `struct`, а также определять их реализацию с помощью `impl`. В `impl` можно реализовать различные типажи, чтобы расширить ваш тип дополнительной функциональностью. Вы также можете ограничивать типы, позволяя делать вызовы только определенным типам, реализующим что-то. Давайте возьмем пример.

В Rust есть много разных типов строк. Есть `str`, `String`, `OsStr`, `OsString`, `CString` и `CStr` (я что-нибудь пропустил?). Практически, обычными являются `str` и `String`, а другие - это для специального назначения. Если мы объявим функцию, в которую мы хотим принимать строку, какой тип мы должны использовать?

Тип `&str` является хорошим выбором, можно передать `&str` или даже `&String` и это будет работать, потому что `String` реализует типаж `Deref<Target=str>` (синтаксис `Target=` означает, что `Target` является "ассоциированным типом"). Но мы могли бы быть более обобщенными с помощью полиморфного типа.

```rust
fn foo<S: AsRef<str>>(s: S) {
    unimplemented!()
}
```

`AsRef` - это типаж для преобразования в библиотеке [stdlib](https://doc.rust-lang.org/std/convert/trait.AsRef.html) . Он определяется как:

```rust
trait AsRef<T>
  where
    T: ?Sized,
{
    fn as_ref(&self) -> &T;
}
```

Это типаж, который принимает параметр типа T, который *может* быть не размерным и определяет функцию, которая превращает этот тип в тип `&T`. Если мы вызываем нашу оригинальную функцию `foo` с типом `String`,

```rust
let a = String::from("stuff");
foo(a);
```

Почему это работает? Посмотрите, сможете ли вы понять это сами (если что смотрите [подсказку](https://doc.rust-lang.org/std/string/struct.String.html#implementations) ). При написании идиоматичного Rust кода важно знать встроенные [типажи преобразования](https://stevedonovan.github.io/rustifications/2018/09/08/common-rust-traits.html) доступные в stdlib.

Я могу привести множество других примеров, но этот пост уже слишком длинный. По сути, типажи используются для инкапсуляции поведения, которое может иметь тип. Тип реализует эти поведения, чтобы получить набор этих функций. Во многих отношениях он работает как интерфейс в Java (тем более, что Java представила реализации по умолчанию в интерфейсах). Но Java естественным образом подходит стилю, ориентированному на наследование. В мире ОО популярно повторять мантру «композиция над наследованием». В Rust у вас нет выбора, это полностью композиция.

## Итог

Есть много тем, которые я мог бы охватить в этой статье, я действительно не знал с чего начать. Надеюсь, я хотя бы что-то осветил для начинающих. Не стесняйтесь обращаться ко мне, если у вас есть идеи для новых вещей, которые вы хотели бы исследовать. Если вы только начинаете с Rust и чувствуете себя немного перегруженным, не волнуйтесь! Все эти темы могут быть новыми и разными, единственное лекарство это чтение и написание большого количества Rust кода.

До скорого.

#### Примечание: про обощенные типы в Java по сравнению с Rust

Когда Java был впервые выпущен, он не включал реализацию обобщений. Этот функционал был высоко оценен, потому что позволял повысить безопасность типов и мог убрать некоторые явные приведения. Байт-код Java не имеет концепции параметра обобщенного типа, т.к. было важно поддерживать обратную совместимость. Таким образом, после того, как компилятор Java подтвердит, что все обобщенные ограничения удовлетворены (указанные в Java аналогично `<T>` или `<T extends Class>` ), Java выполняет то, что называется [стиранием типа (type erasure)](https://docs.oracle.com/javase/tutorial/java/generics/erasure.html). Основы удаления типа заключаются в том, что все параметры типа заменены на `Object` и поэтому никак не ограничены в конечном байт-коде. Я не собираюсь критиковать выбор реализации в Java, но достаточно сказать, что есть один конкретный недостаток о котором я хочу упомянуть - косвенность. Из-за стирания типов общие аргументы передаются в виде указателей на vtable, потому что во время компиляции мы потеряли некоторую информацию о конкретном типе аргумента.

Реализация обобщенных типов в Rust не использует стирание типов. В приведенном выше методе `foo` для каждого случая вызывающего метод `foo` с использованием отдельного конкретного типа, новая версия будет сгенерирована для `foo` и скомпилирована. Это означает, что если `foo` вызвана с 4 различными реализациями `AsRef<str>`, мы потенциально можем получить 4 разные версии функции в нашем финальном коде. Этот процесс называется «мономорфизация». Основным преимуществом этого подхода является то, что он быстрый и Rust хочет предоставлять "абстракции с нулевой стоимостью". Все обобщенные вызовы являются статически диспетчиризируемыми (если явно не обернуто в  `Box`), нам не нужно выделять память в куче для нового объекта `Object` или передавать виртуальную таблицу, если мы этого не хотим. Следует отметить, что недостатком этого способа является конечный размер кода и время компиляции. В зависимости от того, сколько различных вариантов наших функций и сколько реализаций нам нужно сгенерировать, тем больше кода должно пройти через LLVM, увеличивая время компиляции и увеличивая объем фактически создаваемого кода.

- [mail](mailto:cameron.evan@gmail.com "Email me")<br>{a1 href="https://github.com/leshow"}github{/a1}<br>{a2 href="https://twitter.com/evan_cam_"}twitter{/a2}<br>{a3 href="https://linkedin.com/in/evan-cameron"}linkedIn{/a3}<br>{a4 href="leshow.github.io"}Evan Cameron{/a4}
- [github](https://github.com/leshow "GitHub")
- [twitter](https://twitter.com/evan_cam_ "Twitter")
- [linkedIn](https://linkedin.com/in/evan-cameron "LinkedIn")

[Evan Cameron](leshow.github.io)  •  2020  •  [Esoterically Typed](https://leshow.github.io)