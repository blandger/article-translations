# Как я написал современную С++ библиотеку используя Rust

автор [Henri Sivonen](https://hsivonen.fi/author/), оригинальная статья [опубликована 2018-12-03](https://hsivonen.fi/modern-cpp-in-rust/)

Начиная с версии Firefox 56 получил новую версию библиотеку кодирования и конверации символов encoding_rs. Она написана на Rust и заменила старую C++ библиотеку uconv датированую 1999 годом. В начале все вызовы новой библиотеки конвертации были из C++ кода. Поэтому новая библиотека, не смотря на то что написана на Rust, нуждалась в адаптации при вызовах из C++ кода. На самом деле библиотека выглядит для вызывающего из C++ кода как современная C++ библиотека. Вот подходы, которые я использовал чтобы добиться этого.

Есть еще один [блог-пост на английском](https://hsivonen.fi/encoding_rs/) про саму библиотеку. Я представил материал данной статьи на докладе **RustFest Paris** ([видео](https://media.ccc.de/v/rustfest18-5-a_rust_crate_that_also_quacks_like_a_modern_c_library) , [слайды](https://hsivonen.fi/rustfest2018/)) 
 
 ## Современный C++ в каком смысле?
 
 “Современный” C++ в моем случае означает что интерфейс, который видят вызывающие из C++, соответствует [C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines) и использует новые определенные функции:
 
 * Выделение хип-памяти управляется возвращаемыми указателями на хип-объекты внутри std::unique_ptr / mozilla::UniquePtr.
 * Буферы памяти выделенные вызывающей стороной представлены используя gsl::span / mozilla::Span вместо обычных указателей и длины.
 * Возвращаемые значения представлены используя std::tuple / mozilla::Tuple вместо выходных "out" параметров.
 * Не нулевые (non-null) простые указатели аннотированы используя gsl::not_null / mozilla::NotNull.   
 
Упомянутые выше gsl:: это библиотека [Guidelines Support Library](https://github.com/microsoft/GSL), которая предоставляет функции, появление которых ожидается в Core Guidelines, но они еще не доступны в стандартной C++ библиотеке.

## C++ библиотека на Rust?

Написав библиотека C++ “на Rust” я говорю, что основная часть функционала написана на Rust, но интерфейсы предоставленные для вызова из C++ кода позволяют ей выглядеть, как будто она настоящая C++ библиотека.

## C++ и Rust совместимы через C Interop

C++ имеет достаточно сложный ABI ([Двоичный (бинарный) интерфейс приложений](https://ru.wikipedia.org/wiki/%D0%94%D0%B2%D0%BE%D0%B8%D1%87%D0%BD%D1%8B%D0%B9_%D0%B8%D0%BD%D1%82%D0%B5%D1%80%D1%84%D0%B5%D0%B9%D1%81_%D0%BF%D1%80%D0%B8%D0%BB%D0%BE%D0%B6%D0%B5%D0%BD%D0%B8%D0%B9)), ABI в Rust не стабилен и не документирован. Тем не менее для совместимости между C++ и Rust оба ЯП поддерживают функции, которые используют стабильный ABI языка Cи. Поэтому, создание совместимости между C++ и Rust включает написание некоторых вещей таким образом, что C++ "видит" Rust как Cи код, Rust "видит" C++ код тоже как Cи код.

## Упрощающие факторы

Данное описание не следует считать обстоятельным руководством по экспорту Rust кода на C++. Интерфейс к encoding_rs достаточно простой, в нем нет сложностей, которые можно ожидать в общем случае при взаимодействии между двумя языками. Тем не менее факторы, которые упрощают предоставление функций из encoding_rs в C++, можно использовать как небольшое руководство для упрощения необходимое в интересах более простой совместимости между языками при проектировании библиотек. Особенно обратите внимание на:

* encoding_rs никогда не вызывает C++, вызовы между языками однонаправленные
* encoding_rs не держит ссылки на объекты C++ после возврата из вызова, нет необходимости в Rust коде управлять памятью C++
* encoding_rs не представляет иерархию наследований ни в коде Rust ни в C++, отсутствуют vtables в обоих частях.
* Типы данных которыми манипулирует encoding_rs очень простые, последовательные буфферы и примитивы (buffers of u8/uint8_t и u16/char16_t). 
* Есть только поддержка конфигурации panic=abort (т.е. паника в Rust останавливает программу вместо разворачивания стэка) и представленный код является корректным если только использована данная опция. Представленный код не пытается предотвращать панику в Rust panics и разварачивать стэк вызова через FFI, т.к. это будет означать Undefined Behavior. ([Данное поведение было изменено](https://github.com/rust-lang/rust/pull/55982))

## Краткий обзор API

Чтобы получить представление [обсуждаемое Rust API](https://docs.rs/encoding_rs/0.8.14/encoding_rs/), сделаем его высоко-уровневый обзор. Библиотека имеет три публичных структуры: Encoding, Decoder и Encoder. С точки зрения пользователя библиотеки, эти три структуры используются как трейты (traits),  супер-классы (superclasses) или интерфейсы (interfaces) в том смысле, что предоставляют универсальный интерфейс к конкретным реализациям кодировки (encodings), но технически они являются конечно структурами. Экземпляры кодировок (Encoding) аллоцируются статически. Decoder и Encoder содержат состояние кодирования потока и память для них выделяется во время выполнения (at run-time).

```rust
let encoding: &'static Encoding =
    Encoding::for_label( // by label
        byte_slice_from_protocol
    ).unwrap_or(
        WINDOWS_1252     // by named static
    );

let decoder: Decoder = encoding.new_decoder();
```
В случае обработки потока, метод декодирования данных из среза (slice) аллоцированного вызывающей стороной в другой срез (slice) также аллоцированный вызывающим кодом доступен в коде Decoder. Декодер не выполняет выделение хип-памати.  

```rust
pub enum DecoderResult {
    InputEmpty,
    OutputFull,
    Malformed(u8, u8),
}

impl Decoder {
    pub fn decode_to_utf16_without_replacement(
        &mut self,
        src: &[u8],
        dst: &mut [u16],
        last: bool
    ) -> (DecoderResult, usize, usize)
}
```
Когда поток не обрабатывается, вызывающему коду совсем нет необходимости работать с Decoder и Encoder. Вместо этого предоставлены методы для обработки всего логического входного потока в виде одного буфера в компоненте Encoding. 

```rust
impl Encoding {
    pub fn decode_without_bom_handling_and_without_replacement<'a>(
        &'static self,
        bytes: &'a [u8],
    ) -> Option<Cow<'a, str>>
}
```

## Процесс

### Дизайн дружественного FFI

Некоторые факторы для упрощения появляются из самой предметной области, другие факторы дело выбора.

Библиотека кодирования символов могла быть представлена "трейтами" (аналогично абстрактным супер-классам без полей в C++) для каждой из концепций - encoding, decoder и encoder. Вместо этого, encoding_rs имеет для этого структуры, которые своим внутренним представлением совпадают с перечислениями (enum) для диспатча статических вызовов вместо использования динамического вызова на основе [vtable](https://en.wikipedia.org/wiki/Virtual_method_table).

```rust
pub struct Decoder { // no vtable
   variant: VariantDecoder,
   // ...
}

enum VariantDecoder { // no extensibility
    SingleByte(SingleByteDecoder),
    Utf8(Utf8Decoder),
    Gb18030(Gb18030Decoder),
    // ...
}
```

Основной мотивацией для этого было не столько устранение виртуальных таблиц как таковых, сколько намерение сделать иерархию нерасширяемой. Это отражает философию, согласно которой добавление кодировок символов - это не то что должны делать сами программисты. Вместо этого программы должны использовать UTF-8 для внутренного обмена. И программы должны поддерживать устаревшие кодировки только в той степени, которая необходима для совместимости с существующим (legacy) контентом. Нерасширяемость иерархии обеспечивает большую безопасность типов в данном случае. Если у вас есть кодировка из библиотеки encoding_rs, вы можете верить, что она не выставляет кодировки, которые не представлены и не определенны в стандарте кодирования. То есть вы можете доверять, что она не будет вести себя как UTF-7 или EBCDIC.

Дополнительно, с помощью вызова через энум (enum), декодер для одного кодирования может внутренне преобразовываться для в декодер другой кодировки на основе слежения и анализа [BOM](https://en.wikipedia.org/wiki/Byte_order_mark).  

Можно говорить о том, что Rust-овый способ предоставления конвертора кодировок заключался бы в некотором превращении их в "адаптеры итераторов", которые принимают итератор байтов, возвращая скалярные значения в Unicode и наоборот. В дополнение к тому, что итераторы более сложны для предоставления через FFI, итераторы усложняют выполнение трюков для ускорения обработки ASCII. Взятие строкового слайса (slice) для чтения и слайса для записи, не только упрощает представление объектов в C API (в терминах Cи, Rust слайс разбивается на выровненный ненулевой указатель и длину), но также делает возможным ускорение ASCII путем одновременной обработки более чем одной кодовой единицы за раз, делая возможным знаний о том, что несколько кодовых единиц помещаются в один регистр (регистр ALU или регистр SIMD).

Если Rust-native API имеет дело только с примитивами, слайсами и (не трейт объектными) структурами, то проще сопоставить их с Cи API, чем с Rust API, который имеет дело с более причудливыми возможностями Rust. (В Rust у вас есть трейт-объект, когда происходит стирание типа (type erasure). То есть, у вас есть трейт-типизированная ссылка, которая не сообщает конкретный тип структуры референта, реализующий данный трейт.)

### 1. Создание Cи API

When the types involved are simple enough, the main mismatches between C and Rust are the lack of methods and multiple return values in C and the inability to transfer non-C-like structs by value.

*   Methods are wrapped by functions whose first argument is a pointer to the struct whose method is being wrapped.
*   Slice arguments become two arguments: the pointer to the start of the slice and the length of the slice.
*   One primitive value is returned as a function return value and the rest become out params. When the output params clearly relate to inputs of the same type, it makes sense to use in/out params.
*   When a Rust method returns the struct by value, the wrapper function boxes it and returns a pointer such that the Rust side forgets about the struct. Additionally, a function for freeing a given struct type by pointer is added. Such a method simply turns pointer back into a box and drops the box. The struct is opaque from the C point of view.
*   As a special case, the method for getting the name of an encoding, which in Rust would return `&'static str` is wrapped by a function that takes a pointer to writable buffer whose length must be at least the length of the longest name.
*   `enum`s signaling the exhaustion of the input buffer, the output buffer becoming full or errors with detail about the error became `uint32_t` with constants for “input empty” and “output full” and rules for how to interpret the other error details. This isn’t ideal but works pretty well in this case.
*   Overflow-checking length computations are presented as saturating instead. That is, the caller has to treat `SIZE_MAX` as a value signaling overflow.

### 2\. Re-Creating the Rust API in C++ over the C API

Even [an idiomatic C API](https://github.com/hsivonen/encoding_c/blob/master/include/encoding_rs.h) doesn’t make for a modern C++ API. Fortunately, Rustic concepts like multiple return values and slices can be represented in C++, and by reinterpreting pointers returned by the C API as pointers to C++ objects, it’s possible to present the ergonomics of C++ methods.

Most of the examples are from [a version of the API that uses C++17 standard library types](https://github.com/hsivonen/encoding_c/blob/master/include/encoding_rs_cpp.h). In Gecko, we generally avoid the C++ standard library and use [a version of the C++ API to encoding_rs that uses Gecko-specific types](https://searchfox.org/mozilla-central/source/intl/Encoding.h). I assume that the standard-library-type examples make more sense to a broader audience.

#### Method Ergonomics

For each opaque struct pointer type in C, a class is defined in C++ and the C header is tweaked such that the pointer types become pointers to instances of the C++ classes from the point of view of the C++ compiler. This amounts to a `reinterpret_cast` of the pointers without actually writing out the `reinterpret_cast`.

Since the pointers don’t truly point to instances of the classes that they appear to point to but point to instances of Rust structs instead, it’s a good idea to take some precautions. No fields are declared for the classes. The default no-argument and copy constructors are `delete`d as is the default `operator=`. Additionally, there must be no virtual methods. (This last point is an important limitation that will come back to later.)

    class Encoding final {
    // ...
    private:
        Encoding() = delete;
        Encoding(const Encoding&) = delete;
        Encoding& operator=(const Encoding&) = delete;
        ~Encoding() = delete;
    };

In the case of `Encoding` whose all instances are static, the destructor is `delete`d as well. In the case of the dynamically-allocated `Decoder` and `Encoder` both an empty destructor and a `static void operator delete` is added. (An example follows a bit later.) This enables the destruction of the fake C++ class to be routed to the right type-specific freeing function in the C API.

With that foundation in place to materialize pointers that look like pointers to C++ class instances, it’s possible to make method calls on this pointers work. (An example follows after introducing the next concept, too.)

#### Returning Dynamically-Allocated Objects

As noted earlier, the cases where the Rust API would return an `Encoder` or a `Decoder` by value so that the caller can place them on the stack is replaced by the FFI wrapper boxing the objects so that the C API exposes only heap-allocated objects by pointer. Also, the reinterpretation of these pointers as `delete`able C++ object pointers was already covered.

That still leaves making sure that `delete` is actually used at an appropriate time. In modern C++, when an object can have only one legitimate owner of the time, this is accomplished by wrapping the object pointer in `std::unique_ptr` or `mozilla::UniquePtr`. The old `uconv` converters supported reference counting, but all the actual uses in the Gecko code base involved only one owner for each converter. Since the usage patterns of encoders and decoders are such that there is only one legitimate owner of the time, using `std::unique_ptr` and `mozilla::UniquePtr` is what the two C++ wrappers for encoding_rs do.

Let’s take a look at a factory method on `Encoding` that returns a `Decoder`. In Rust, we have a method that takes a reference to `self` and returns `Decoder` by value.

    impl Encoding {
        pub fn new_decoder(&'static self) -> Decoder {
            // ...
        }
    }

On the FFI layer, we have an explicit pointer-typed first argument that corresponds to Rust `&self` and C++ `this` (specifically, the `const` version of `this`). We allocate memory on the heap (`Box::new()`) and place the `Decoder` into the allocated memory. We then forget about the allocation (`Box::into_raw`) so that we can return the pointer to C without deallocating at the end of the scope. In order to be able to free the memory, we introduce a new function that puts the `Box` back together and assigns it into a variable that immediately goes out of scope causing the heap allocation to be freed.

    #[no_mangle]
    pub unsafe extern "C" fn encoding_new_decoder(
        encoding: *const Encoding) -> *mut Decoder
    {
        Box::into_raw(Box::new((*encoding).new_decoder()))
    }

    #[no_mangle]
    pub unsafe extern "C" fn decoder_free(decoder: *mut Decoder) {
        let _ = Box::from_raw(decoder);
    }

In the C header, they look like this:

    ENCODING_RS_DECODER*
    encoding_new_decoder(ENCODING_RS_ENCODING const* encoding);

    void
    decoder_free(ENCODING_RS_DECODER* decoder);

`ENCODING_RS_DECODER` is a macro that is used for substituting the right C++ type when the C header is used in the C++ context instead of being used as a plain C API.

On the C++ side, then, we use `std::unique_ptr`, which is the C++ analog of Rust’s `Box`. They are indeed very similar:

<dl>

<dt>`<span class="hljs-keyword">let</span> ptr: <span class="hljs-built_in">Box</span><Foo>`</dt>

<dd>`<span class="hljs-built_in">std</span>::<span class="hljs-built_in">unique_ptr</span><Foo> ptr`</dd>

<dt>`<span class="hljs-built_in">Box</span>::new(Foo::new(a, b, c))`</dt>

<dd>`make_unique<Foo>(a, b, c)`</dd>

<dt>`<span class="hljs-built_in">Box</span>::into_raw(ptr)`</dt>

<dd>`ptr.release()`</dd>

<dt>`<span class="hljs-keyword">let</span> ptr = <span class="hljs-built_in">Box</span>::from_raw(raw_ptr);`</dt>

<dd>`<span class="hljs-built_in">std</span>::<span class="hljs-built_in">unique_ptr</span><Foo> ptr(raw_ptr);`</dd>

</dl>

We wrap the pointer obtained from the C API in a `std::unique_ptr`:

    class Encoding final {
    public:
        inline std::unique_ptr<Decoder> new_decoder() const
        {
            return std::unique_ptr<Decoder>(
                encoding_new_decoder(this));
        }
    };

When the `std::unique_ptr<Decoder>` goes out of scope, the deletion is routed back to Rust via FFI thanks to declarations like this:

    class Decoder final {
    public:
        ~Decoder() {}
        static inline void operator delete(void* decoder)
        {
            decoder_free(reinterpret_cast<Decoder*>(decoder));
        }
    private:
        Decoder() = delete;
        Decoder(const Decoder&) = delete;
        Decoder& operator=(const Decoder&) = delete;
    };

#### How Can it Work?

In Rust, non-trait methods are just syntactic sugar:

    impl Foo {
        pub fn get_val(&self) -> usize {
            self.val
        }
    }

    fn test(bar: Foo) {
        assert_eq!(bar.get_val(), Foo::get_val(&bar));
    }

A method call on non-trait-typed reference is just a plain function call with the reference to `self` as the first argument. On the C++ side, non-virtual method calls work the same way: A non-virtual C++ method call is really just a function call whose first argument is the `this` pointer.

On the FFI/C layer, we can pass the same pointer as an explicit pointer-typed first argument.

When calling `ptr->Foo()` where `ptr` is of type `T*`, the type of `this` is `T*` if the method is declared as `void Foo()` (which maps to `&mut self` in Rust) and `const T*` if the method is declared as `void Foo() const` (which maps to `&self` in Rust), so `const`-correctness is handled, too.

<dl>

<dt>`<span class="hljs-function"><span class="hljs-keyword">fn</span> <span class="hljs-title">foo</span></span>(&<span class="hljs-keyword">self</span>, bar: <span class="hljs-built_in">usize</span>) -> <span class="hljs-built_in">usize</span>`</dt>

<dd>`<span class="hljs-keyword">size_t</span> foo(<span class="hljs-keyword">size_t</span> bar) <span class="hljs-keyword">const</span>`</dd>

<dt>`<span class="hljs-function"><span class="hljs-keyword">fn</span> <span class="hljs-title">foo</span></span>(&<span class="hljs-keyword">mut</span> <span class="hljs-keyword">self</span>, bar: <span class="hljs-built_in">usize</span>) -> <span class="hljs-built_in">usize</span>`</dt>

<dd>`<span class="hljs-keyword">size_t</span> foo(<span class="hljs-keyword">size_t</span> bar)`</dd>

</dl>

The qualifications about “non-trait-typed” and “non-virtual” are important. For the above to work, _we can’t have vtables on either side_. This means no Rust trait objects and no C++ inheritance. In Rust, trait objects, i.e. trait-typed references to any struct that implements the trait, are implemented as two pointers: one to the struct instance and another to the vtable appropriate for the concrete type of the data. We need to be able to pass reference to `self` across the FFI as a single pointer, so there’s no place for the vtable pointer when crossing the FFI. In order to keep pointers to C++ objects as C-compatible plain pointers, C++ puts the vtable pointer on the objects themselves. Since the pointers don’t really point to C++ objects carrying vtable pointers but point to Rust objects, we must make sure not to make the C++ implementation expect to find a vtable pointer on the pointee.

As a consequence, the C++ reflector classes for the Rust structs cannot inherit from a common baseclass of a C++ framework. In the Gecko case, the reflector classes cannot inherit from `nsISupports`. E.g. in the context of Qt, the reflector classes wouldn’t be able to inherit from `QObject`.

#### Non-Nullable Pointers

There are methods in the Rust API that return `&'static Encoding`. Rust references can never be null, and it would be nice to relay this piece of information in the C++ API. It turns out that there is a C++ idiom for this: `gsl::not_null` and `mozilla::NotNull`.

Since `gsl::not_null` and `mozilla::NotNull` are just type system-level annotations that don’t change the machine representation of the underlying pointer and since from the guarantees Rust we know which pointers that we get from the FFI really never are null, it is tempting to apply the same reinterpretation trick of lying to the C++ compiler about types that we use to reinterpret pointers returned by the FFI as pointers to fieldless C++ objects with no virtual methods and to claim in a header file that the pointers that we know not to be null in the FFI return values are of the type `mozilla::NotNull<const Encoding*>`. Unfortunately, this doesn’t actually work because types involving templates are not allowed in the declarations of `extern "C"` functions in C++, so the C++ code ends up executing a branch for the null check when wrapping pointers received from the C API with `gsl::not_null` or `mozilla::NotNull`.

However, there are also declarations of static pointers to the constant encoding objects (where the pointees are defined in Rust) and it happens that C++ _does_ allow declaring those as `gsl::not_null<const Encoding*>`, so that is what is done. (Thanks to Masatoshi Kimura for pointing out that this is possible.)

The statically-allocated instances of `Encoding` are declared in Rust like this:

    pub static UTF_8_INIT: Encoding = Encoding {
        name: "UTF-8",
        variant: VariantEncoding::Utf8,
    };

    pub static UTF_8: &'static Encoding = &UTF_8_INIT;

In Rust, the [general rule](https://twitter.com/tshepang_dev/status/1051558270425591808) is that you use `static` for an unchanging memory location and `const` for an unchanging value. Therefore, `UTF_8_INIT` should be `static` and `UTF_8` should be `const`: the value of the reference to the `static` instance is unchanging, but statically allocating a memory location for the reference is not logically necessary. Unfortunately, Rust has a rule that says that the right-hand side of `const` may not contain anything `static` and this is applied so heavily as to prohibit even references to `static`, in order to ensure that the right-hand side of a `const` declaration can be statically checked to be suitable for use within any imaginable `const` declaration—even one that tried to dereference the reference at compile time.

For FFI, though, we need to allocate an unchanging memory location to a pointer to `UTF_8_INIT`, because such memory locations work in C linkage and allow us provide a pointer-typed named thing to C. The representation of `UTF_8` above is already what we need, but for Rust ergonomics, we want `UTF_8` to participate in Rust’s crate namespacing. This means that from the C perspective the name gets mangled. We waste some space by statically allocating pointers _again_ without name mangling for C usage:

    pub struct ConstEncoding(*const Encoding);

    unsafe impl Sync for ConstEncoding {}

    #[no_mangle]
    pub static UTF_8_ENCODING: ConstEncoding =
        ConstEncoding(&UTF_8_INIT);

A pointer type is used to make in clear that C is supposed to see a pointer (even if a Rust reference type would have the same representation). However, the Rust compiler refuses to compile a program with globally-visible pointer. Since globals are reachable from different threads, multiple threads accessing the pointee might be problem. In this case, the pointee cannot be mutated, so global visibility is fine. To tell the compiler that this is fine, we need to implement the `Sync` marker trait for the pointer. However, traits cannot be implemented on pointer types. As a workaround, we create a newtype for `*const Encoding`. A newtype has the same representation as the type it wraps, but we can implement traits on the newtype. Implementing `Sync` is `unsafe`, because we are asserting to the compiler that something is OK when the compiler does not figure it out on its own.

In C++, we can then say (what via macros expands to):

    extern "C" {
        extern gsl::not_null<const encoding_rs::Encoding*> const UTF_8_ENCODING;
    }

The pointers to the encoders and decoders are also known not to be null, since allocation failure would terminate the program, but `std::unique_ptr` / `mozilla::UniquePtr` and `gsl::not_null` / `mozilla::NotNull` cannot be combined.

#### Optional Values

In Rust, it’s idiomatic to use `Option<T>` to represent return values might either have a value or might not have a value. C++ these days provides the same thing as `std::optional<T>`. In Gecko, we instead have `mozilla::Maybe<T>`.

Rust’s `Option<T>` and C++’s `std::optional<T>` indeed are basically the same thing:

<dl>

<dt>`<span class="hljs-keyword">return</span> <span class="hljs-literal">None</span>;`</dt>

<dd>`<span class="hljs-keyword">return</span> <span class="hljs-built_in">std</span>::nullopt;`</dd>

<dt>`<span class="hljs-keyword">return</span> <span class="hljs-literal">Some</span>(foo);`</dt>

<dd>`<span class="hljs-keyword">return</span> foo;`</dd>

<dt>`is_some()`</dt>

<dd>`<span class="hljs-function"><span class="hljs-keyword">operator</span> <span class="hljs-title">bool</span><span class="hljs-params">()</span></span>`</dd>

<dd>`has_value()`</dd>

<dt>`unwrap()`</dt>

<dd>`value()`</dd>

<dt>`unwrap_or(bar)`</dt>

<dd>`value_or(bar)`</dd>

</dl>

Unfortunately, though, C++ reverses the safety ergonomics. The most ergonomic way to extract the wrapped value from a `std::optional<T>` is via `operator*()`, which is unchecked and, therefore, unsafe. <span class="emoji">😭</span>

#### Multiple Return Values

While C++ lacks language-level support for multiple return values, multiple return values are possible thanks to library-level support. In the case of the standard library, the relevant library pieces are `std::tuple`, `std::make_tuple` and `std::tie`. In the case of Gecko, the relevant library pieces are `mozilla::Tuple`, `mozilla::MakeTuple` and `mozilla::Tie`.

<dl>

<dt>`<span class="hljs-function"><span class="hljs-keyword">fn</span> <span class="hljs-title">foo</span></span>() -> (T, U, V)`</dt>

<dd>`<span class="hljs-built_in">std</span>::tuple<T, U, V> foo()`</dd>

<dt>`<span class="hljs-keyword">return</span> (a, b, c);`</dt>

<dd>`<span class="hljs-keyword">return</span> {a, b, c};`</dd>

<dt>`<span class="hljs-keyword">let</span> (a, b, c) = foo();`</dt>

<dd>`<span class="hljs-keyword">const</span> <span class="hljs-keyword">auto</span> [a, b, c] = foo();`</dd>

<dt>`<span class="hljs-keyword">let</span> <span class="hljs-keyword">mut</span> (a, b, c) = foo();`</dt>

<dd>`<span class="hljs-keyword">auto</span> [a, b, c] = foo();`</dd>

</dl>

#### Slices

A Rust slice wraps a non-owning pointer and a length that identify a contiguous part of an array. In comparison to C:

<dl>

<dt>`src: &[<span class="hljs-built_in">u8</span>]`</dt>

<dd>`<span class="hljs-keyword">const</span> <span class="hljs-keyword">uint8_t</span>* src, <span class="hljs-keyword">size_t</span> src_len`</dd>

<dt>`dst: &<span class="hljs-keyword">mut</span> [<span class="hljs-built_in">u8</span>]`</dt>

<dd>`<span class="hljs-keyword">uint8_t</span>* dst, <span class="hljs-keyword">size_t</span> dst_len`</dd>

</dl>

There isn’t a corresponding thing in the C++ standard library yet (except `std::string_view` for read-only string slices), but it’s already part of the C++ Core Guidelines and is called a [span](https://github.com/isocpp/CppCoreGuidelines/blob/master/CppCoreGuidelines.md#i13-do-not-pass-an-array-as-a-single-pointer) there.

<dl>

<dt>`src: &[<span class="hljs-built_in">u8</span>]`</dt>

<dd>`gsl::span<<span class="hljs-keyword">const</span> <span class="hljs-keyword">uint8_t</span>> src`</dd>

<dt>`dst: &<span class="hljs-keyword">mut</span> [<span class="hljs-built_in">u8</span>]`</dt>

<dd>`gsl::span<<span class="hljs-keyword">uint8_t</span>> dst`</dd>

<dt>`&<span class="hljs-keyword">mut</span> vec[..]`</dt>

<dd>`gsl::make_span(vec)`</dd>

<dt>`std::slice::from_raw_parts(ptr, len)`</dt>

<dd>`gsl::make_span(ptr, len)`</dd>

<dt>`<span class="hljs-keyword">for</span> item <span class="hljs-keyword">in</span> slice {}`</dt>

<dd>`<span class="hljs-keyword">for</span> (<span class="hljs-keyword">auto</span>&& item : span) {}`</dd>

<dt>`slice[i]`</dt>

<dd>`span[i]`</dd>

<dt>`slice.len()`</dt>

<dd>`span.size()`</dd>

<dt>`slice.as_ptr()`</dt>

<dd>`span.data()`</dd>

</dl>

GSL relies on C++14, but at the time encoding_rs landed, Gecko was [stuck on C++11 thanks to Android](https://bugzilla.mozilla.org/show_bug.cgi?id=1325632#c25). Since, GSL could not be used as-is in Gecko, I backported `gsl::span` to C++11 as [`mozilla::Span`](https://searchfox.org/mozilla-central/source/mfbt/Span.h#375). The porting process was mainly a matter of ripping out `constexpr` keywords and using `mozilla::` types and type traits in addition to or instead of standard-library ones. After Gecko moved to C++14, some of the `constexpr` keywords have been restored.

Once we had our own `mozilla::Span` anyway, it was possible to add Rust-like subspan ergonomics that are missing from `gsl::span`. For the case where you want a subspan from index `i` up to but not including index `j`. `gsl::span` has:

<dl>

<dt>`&slice[i..]`</dt>

<dd>`span.subspan(i)`</dd>

<dt>`&slice[..i]`</dt>

<dd>`span.subspan(<span class="hljs-number">0</span>, i)`</dd>

<dt>`&slice[i..j]`</dt>

<dd>`span.subspan(i, j - i)` <span class="emoji">😭</span></dd>

</dl>

`mozilla::Span` instead has:

<dl>

<dt>`&slice[i..]`</dt>

<dd>`span.From(i)`</dd>

<dt>`&slice[..i]`</dt>

<dd>`span.To(i)`</dd>

<dt>`&slice[i..j]`</dt>

<dd>`span.FromTo(i, j)`</dd>

</dl>

`gsl::span` and Rust slices have one crucial difference in how they decompose into a pointer and a length. For zero-length `gsl::span` it is possible for the pointer to be `nullptr`. In the case of Rust slices, the pointer must always be non-null and aligned even for zero-length slices. This may look counter-intuitive at first: When the length is zero, the pointer never gets dereferenced, so why doesn’t matter whether it is null are not? It turns out that it matters for optimizing out the `enum` discriminant in `Option`-like enums. `None` is represented by all-zero bits, so if wrapped in `Some()`, a slice with null as the pointer and zero as the length would accidentally have the same representation as `None`. By requiring the pointer to be a potentially bogus non-null pointer, a zero-length slice inside an `Option` can be represented distinctly from `None` without a discriminant. By requiring the pointer to be aligned, further uses of the low bits of the pointer are possible when the alignment of the slice element type is greater than one.

After realizing that it’s not okay to pass the pointer obtained from C++ `gsl::span::data()` to Rust `std::slice::from_raw_parts()` as-is, it was necessary to decide where to put the replacement of `nullptr` with `reinterpret_cast<T*>(alignof(T))`. There are two candidate locations when working with actual `gsl::span`: In the Rust code that provides the FFI or in the C++ code that calls the FFI. When working with `mozilla::Span`, the code of the span implementation itself could be changed, so there are two additional candidate locations for the check: the constructor of `mozilla::Span` and the getter for the pointer.

Of these for candidate locations, the constructor of `mozilla::Span` seemed like the one where the compiler has the best opportunity to optimize out the check in some cases. That’s why I chose to put the check there. This means that in the `gsl::span` scenario the check had to go in the code that calls the FFI. All pointers obtained from `gsl::span` have to be laundered through:

    template <class T>
    static inline T* null_to_bogus(T* ptr)
    {
        return ptr ? ptr : reinterpret_cast<T*>(alignof(T));
    }

Additionally, this means that since the check is not in the code that provides the FFI, the C API became slightly unidiomatic in the sense that requires C callers to avoid passing in `NULL` even when the length is zero. However, the C API already has many caveats about things that are Undefined Behavior, and adding yet another thing that is documented to be Undefined Behavior does seem like an idiomatic thing to do with C.

#### Putting it Together

Let’s look at an example of how the above features combine. First, in Rust we have a method that takes a slice and returns an optional tuple:

    impl Encoding {
        pub fn for_bom(buffer: &[u8]) ->
            Option<(&'static Encoding, usize)>
        {
            if buffer.starts_with(b"\xEF\xBB\xBF") {
                Some((UTF_8, 3))
            } else if buffer.starts_with(b"\xFF\xFE") {
                Some((UTF_16LE, 2))
            } else if buffer.starts_with(b"\xFE\xFF") {
                Some((UTF_16BE, 2))
            } else {
                None
            }
        }
    }

Since this is a static method, there is no reference to `self` and no corresponding pointer in the FFI function. The slice decomposes into a pointer and a length. The length becomes an in/out param that communicates the length of the slice in and the length of the BOM sublice out. The encoding becomes the return value and the encoding pointer being null communicates the Rust `None` case for the tuple.

    #[no_mangle]
    pub unsafe extern "C" fn encoding_for_bom(buffer: *const u8,
                                              buffer_len: *mut usize)
                                              -> *const Encoding
    {
        let buffer_slice =
            ::std::slice::from_raw_parts(buffer, *buffer_len);
        let (encoding, bom_length) =
            match Encoding::for_bom(buffer_slice) {
            Some((encoding, bom_length)) =>
                (encoding as *const Encoding, bom_length),
            None => (::std::ptr::null(), 0),
        };
        *buffer_len = bom_length;
        encoding
    }

In the C header, the signature looks like this:

    ENCODING_RS_ENCODING const*
    encoding_for_bom(uint8_t const* buffer, size_t* buffer_len);

The C++ layer then rebuilds the analog of the Rust API on top of the C API:

    class Encoding final {
    public:
        static inline std::optional<
            std::tuple<gsl::not_null<const Encoding*>, size_t>>
        for_bom(gsl::span<const uint8_t> buffer)
        {
            size_t len = buffer.size();
            const Encoding* encoding =
                encoding_for_bom(null_to_bogus(buffer.data()), &len);
            if (encoding) {
                return std::make_tuple(
                    gsl::not_null<const Encoding*>(encoding), len);
            }
            return std::nullopt;
        }
    };

Here we have to explicitly use `std::make_tuple`, because the implicit constructor doesn’t work when the `std::tuple` is nested inside `std::optional`.

#### Algebraic Types

Early on, we saw that the Rust-side streaming API can return this `enum`:

    pub enum DecoderResult {
        InputEmpty,
        OutputFull,
        Malformed(u8, u8),
    }

C++ now has an analog for Rust `enum`, sort of: `std::variant<Types...>`. In practice, though, `std::variant` is so clunky that it does not make sense to use it when a Rust `enum` is supposed to act in a lightweight way from the point view of ergonomics.

First, the variants in `std::variant` aren’t named. They are identified positionally or by type. Named variants were proposed as proposed as [`lvariant`](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0095r1.html) but did not get accepted. Second, even though duplicate types are permitted, working with them is not practical. Third, there is no language-level analog for Rust’s [`match`](https://doc.rust-lang.org/book/second-edition/ch06-02-match.html). A `match`-like mechanism was proposed as [`inspect()`](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0095r1.html) but was not accepted.

On the FFI/C layer, the information from the above `enum` is packed into a `u32`. Instead of trying to expand it to something fancier on the C++ side, the C++ API uses the same `uint32_t` as the C API. If the caller actually cares about extracting the two small integers in the malformed case, it’s up to the caller to do the bitwise ops to extract them from the `uint32_t`.

The FFI code looks like this:

    pub const INPUT_EMPTY: u32 = 0;

    pub const OUTPUT_FULL: u32 = 0xFFFFFFFF;

    fn decoder_result_to_u32(result: DecoderResult) -> u32 {
        match result {
            DecoderResult::InputEmpty => INPUT_EMPTY,
            DecoderResult::OutputFull => OUTPUT_FULL,
            DecoderResult::Malformed(bad, good) =>
                (good as u32) << 8) | (bad as u32),
        }
    }

Using zero as the magic value for `INPUT_EMPTY` is a premature micro-optimization. On some architectures comparison with zero is cheaper than comparison with other constants, and the values representing the malformed case when decoding and the unmappable case when encoding are known not to overlap zero.

#### Signaling Integer Overflow

`Decoder` and `Encoder` have methods for querying worst-case output buffer size requirement. The caller provides the number of input code units and the method returns the smallest output buffer length, in code units, that guarantees that the corresponding conversion method will not return `OutputFull`.

E.g. when encoding from UTF-16 to UTF-8, calculating the worst case involves multiplication by three. Such a calculation can, at least in principle, result in integer overflow. In Rust, integer overflow is considered safe, because even if you allocate too short a buffer as a result of its length computation overflowing, actually accessing the buffer is bound checked, so the overall result is safe. However, buffer access is not generally bound checked in C or C++, so an integer overflow in Rust can result in memory unsafety in C or C++ if the result of the calculation that overflowed is used for deciding the size of buffers allocated and accessed by C or C++ code. In the case of encoding_rs, even when C or C++ allocates the buffer, the writing is supposed to be performed by Rust code, so it might be OK. However, to be sure, the worst-case calculations provided by encoding_rs used overflow-checking arithmetic.

In Rust, the methods whose arithmetic is overflow-checked return `Option<usize>`. To keep the types of the C API simple, the C API returns `size_t` with `SIZE_MAX` signaling overflow. That is, the C API effectively appears as using saturating arithmetic.

In the C++ API version that uses standard-library types, the return type is `std::optional<size_t>`. In Gecko, we have a wrapper for integer types that provides overflow-checking arithmetic and a validity flag. In the Gecko version of the C++ API, the return type is `mozilla::CheckedInt<size_t>` so that dealing with overflow signaling is uniform with the rest of Gecko code. (Aside: I find it shocking and dangerous that the C++ standard library _still_ does not provide a wrapper similar to `mozilla::CheckedInt` in order to do overflow-checking integer math in a standard-supported Undefined Behavior-avoiding way.)

## Recreating the Non-Streaming API

Let’s look again at the example of a non-streaming API method on `Encoding`:

    impl Encoding {
        pub fn decode_without_bom_handling_and_without_replacement<'a>(
            &'static self,
            bytes: &'a [u8],
        ) -> Option<Cow<'a, str>>
    }

This type inside the `Option` in the return type is `Cow<'a, str>`, which is a type that holds either an owned `String` or a borrowed string slice (`&'a str`) whose data is owned by someone else. The lifetime `'a` of the borrowed string slice is the lifetime of the input slice (`bytes: &'a [u8]`), because in the borrow case the output is actually borrowed from the input.

Mapping this kind of return type to C poses problems. First of all, C does not provide a great way to say that we either have the owned case or we have the borrowed case. Second, C does not have a standard type for heap-allocated strings that know their length and capacity and that can reallocate their buffer when modified. Maybe this could be seen as an opportunity to create a new C type whose buffer is managed by Rust `String`, but then such a type would not fit together with C++ strings. Third, a borrowed string slice in C would be a raw pointer and a length and some documentation that says that the pointer is valid only as long as the input pointer is valid. There would be no language-level safeguards against use-after-free.

The solution is not to provide the non-streaming API on the C layer at all. On the Rust side, the non-streaming API is a convenience API built on top of the streaming API and some validation functions (ASCII validation, UTF-8 validation, ISO-2022-JP ASCII state validation). Instead of trying to provide FFI bindings for the non-streaming API in an inconvenient manner, a similar non-streaming API can be recreated in C++ on top of the streaming API and the validation functions that were suitable for FFI.

While the C++ type system could represent the same kind of structure as Rust’s `Cow<'a, str>` e.g. as `std::variant<std::string_view, std::string>`, such a C++ `Cow` would be unsafe, because the lifetime `'a` would not be enforced by C++. While a `std::string_view` (or `gsl::span`) is (mostly) OK as an argument in C++, as a return type it’s use-after-free waiting to happen. As with C, at best there would be some documentation saying that the output `std::string_view` is valid for as long as the input `gsl::span` is valid.

To avoid use-after-free risk, in the C++ API version that uses C++17 standard-library types, I simply ended up making the C++ `decode_without_bom_handling_and_without_replacement()` always copy and return a `std::optional<std::string>`.

In the case of Gecko though, it’s possible to do better while keeping things safe. Gecko uses [XPCOM strings](https://developer.mozilla.org/en-US/docs/Mozilla/Tech/XPCOM/Guide/Internal_strings), which provide a variety of storage options, notably: dependent strings that (unsafely) borrow storage owned by someone else, auto strings that store short strings in an inline buffer and shared strings that point to heap-allocated reference-counted buffer.

In the case where the buffer to decode is in an XPCOM string that points to a reference-counted heap-allocated buffer and we are decoding to UTF-8 (as opposed to UTF-16), in the cases where we’d borrow in Rust (expect for BOM removal cases), we can instead make the output string point the same reference-counted heap-allocated buffer that the input points to (and increment the reference count). This is indeed what the non-streaming API for `mozilla::Encoding` does.

Compared to Rust, there is a limitation beyond the input string having to use reference-counted storage for the copy avoidance to work: The input must not have the UTF-8 BOM in the cases where the BOM is removed. While Rust can borrow a subslice of the input excluding the BOM, with XPCOM strings just incrementing a reference count only works if the byte content of the input and output is the entirely the same. When the first three bytes need to be omitted, it’s not the entirely the same.

While the C++ API version that uses C++17 standard library types builds the non-streaming API on top of the streaming API in C++, for added safety, the non-streaming part of `mozilla::Encoding` is not actually built on the streaming C++ API in C++ but built on top of the streaming Rust API [in Rust](https://searchfox.org/mozilla-central/source/intl/encoding_glue/src/lib.rs). In Gecko, we have [Rust bindings for XPCOM strings](https://searchfox.org/mozilla-central/source/servo/support/gecko/nsstring/src/lib.rs), so it’s possible to manipulate XPCOM strings from Rust.

## Epilog: Do We Really Need to Hold `Decoder` and `Encoder` by Pointer?

Apart from having to copy in the non-streaming API due to C++ not having a safe mechanism for borrows, it’s a bit disappointing that instantiating `Decoder` and `Encoder` from C++ involves a heap allocation while Rust callers get to allocate these types on the stack. Can we get rid of the heap allocation for C++ users of the API?

The answer is that we could, but to do it properly we’d end up with the complexity of making the C++ build system generate constants by querying them from rustc.

We can’t return a non-C-like struct over the FFI by value, but given a suitably-aligned pointer to enough memory, we can write a non-C-like struct to memory provided by the other side of the FFI. In fact, the API supports this as an optimization of instantiating a new `Decoder` into a heap allocation made by Rust previously:

    #[no_mangle]
    pub unsafe extern "C" fn encoding_new_decoder_into(
        encoding: *const Encoding,
        decoder: *mut Decoder)
    {
        *decoder = (*encoding).new_decoder();
    }

Even though documentation says that `encoding_new_decoder_into()` should only be used with pointers to `Decoder` previously obtained from the API, in the case of `Decoder`, assigning with `=` would be OK even if the memory pointed to by the pointer was uninitialized, because `Decoder` does not implement `Drop`. That is, in C++ terms, `Decoder` in Rust does not have a destructor, so assignment with `=` does not do any clean-up with the assumption that the pointer points to a previous valid `Decoder`.

When writing a Rust struct that implements `Drop` into uninitialized memory, `std::ptr::write()` should be used instead of `=`. `std::ptr::write()` “overwrites a memory location with the given value without reading or dropping the old value”. Perhaps it would set a good example to use `std::ptr::write()` even in the above case, even though it’s not strictly necessary.

When working with a pointer previously obtained from Rust `Box`, the pointer is aligned correctly and points to a sufficiently large piece of memory. If C++ is to allocate stack memory for Rust code to write into, we need to make the C++ code use the right size and alignment. The issue of communicating these two numbers from Rust to C++ is already where things start getting brittle.

The C++ code needs to discover the right size and alignment for the struct. These cannot be discovered by calling FFI functions, because C++ needs to know them at compile time. Size and alignment aren’t just constants that could be written manually in a header file once and forgotten. First of all, they change when the Rust structs change, so just writing them down has the risk of the written-down values getting out of sync with the real requirements as the Rust code changes. Second, the values differ on 32-bit architectures vs. 64-bit architectures. Third, and this is the worst, the alignment can differ from one 32-bit architecture to another. Specifically, the alignment of `f64` is `8` on most targets, like [ARM](https://rust.godbolt.org/z/IE1t4G), [MIPS](https://rust.godbolt.org/z/PBLXeU) and [PowerPC](https://rust.godbolt.org/z/ErGoPf), but the alignment of `f64` is `4` on [x86](https://rust.godbolt.org/z/4jS6lf). If Rust gets an [m68k port](https://lists.llvm.org/pipermail/llvm-dev/2018-August/125325.html), even more [variety of alignments across 32-bit platforms is to be expected](https://bugzilla.mozilla.org/show_bug.cgi?id=1325771#c49).

It seems that the only way to get this right is to get the size and alignment information from rustc as part of the build process before the C++ code is built so that the numbers can be written in a generated C++ header file that the C++ code can then refer to. The simple way to do this would be to have the build system compile and run a tiny Rust program that prints out a C++ header with numbers obtained using [`std::mem::size_of`](https://doc.rust-lang.org/std/mem/fn.size_of.html) and [`std::mem::align_of`](https://doc.rust-lang.org/std/mem/fn.align_of.html). This solution assumes that the build system runs on the architecture that the compilation is targeting, so this solution would break cross-compilation. That’s not good.

We need to extract target-specific size and alignment from a given struct from rustc but without having to run a binary built for the target. [It turns out](https://blog.mozilla.org/nnethercote/2018/11/09/how-to-get-the-size-of-rust-types-with-zprint-type-sizes/) that rustc has a command-line option, `-Zprint-type-sizes`, that prints out the size and alignment of types. Unfortunately, the feature is nightly-only… Anyway, the most correct way to go about this would be to have a build script controlling C++ compilation first invoke rustc with that option, parse out the sizes and aligments of interest, and generate a C++ header file with the numbers as constants.

Or, since overaligning is permitted, we could trust that the struct will not have a SIMD member (alignment 16 for 128-bit vectors) and always align to 8\. We could also check the size on 64-bit platforms, always use that and hope for the best (especially hope that whenever the struct grows in Rust, someone remembers to update the C++-visible size). But hoping for the best in memory matters kind of defeats the point of using Rust.

Anyway, assuming that we have constants `DECODER_SIZE` and `DECODER_ALIGNMENT` available to C++ _somehow_, we can do this:

    class alignas(DECODER_ALIGNMENT) Decoder final
    {
      friend class Encoding;
    public:
      ~Decoder() {}
      Decoder(Decoder&&) = default;
    private:
      unsigned char storage[DECODER_SIZE];
      Decoder() = default;
      Decoder(const Decoder&) = delete;
      Decoder& operator=(const Decoder&) = delete;
      // ...
    };

Notably:

*   Instead of the constructor `Decoder()` being marked `delete`, it is marked `default` but still `private`.
*   `Encoding` is declared as a `friend` to grant it access to the above-mentioned constructor.
*   A `public` default move constructor is added.
*   A single `private` field of type `unsigned char[DECODER_SIZE]` is added.
*   `Decoder` itself is declared with `alignas(DECODER_ALIGNMENT)`.
*   `operator delete` is no longer overloaded.

Then `new_decoder()` on `Encoding` can be written like this (and be renamed `make_decoder` to avoid unidiomatic use of the word “new” in C++):

    class Encoding final
    {
    public:
      inline Decoder make_decoder() const
      {
        Decoder decoder;
        encoding_new_decoder_into(this, &decoder);
        return decoder;
      }
      // ...
    };

And it can be used like this:

    Decoder decoder = input_encoding->make_decoder();

Note that outside the implementation of `Encoder` trying to just declare `Decoder decoder;` without initializing it right away is a compile-time error, because the constructor `Decoder()` is private.

Let’s unpack what’s happening:

*   The array of `unsigned char` provides storage for the Rust `Decoder`.
*   The C++ `Decoder` has no base class, virtual methods, etc., so there are no implementation-supplied hidden members and the address of a `Decoder` is the same as the address of its `storage` member, so we can simply pass the address of `Decoder` itself to Rust.
*   The alignment of `unsigned char` is 1, i.e. unrestricted, so `alignas` on the `Decoder` gets to determine the alignment.
*   The default trivial move constructor `memmove`s the bytes of the `Decoder`, and the Rust `Decoder` is OK to move.
*   The private default no-argument constructor makes it a compile error to try to declare a not-immediately-initialized instance of the C++ `Decoder` outside the implementation of `Encoder`.
*   `Encoder`, however, can instantiate an uninitialized `Decoder` and pass a pointer to it to Rust, so that Rust code can write the Rust `Decoder` instance into the C++-provided memory via the pointer.
