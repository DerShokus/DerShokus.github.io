---
layout: post
title:  "std::function и как он устроен"
date:   2017-10-28
categories: cpp stl
---
### TLTR:
std::function может аллоцировать память в куче, а может и не аллоцировать (стандарт не обязывает). Все зависит от реализации. В этом посте вы найдете разбор того, как это устроенно в CLang и GCC определенных версий.

### Clang version:
*Следует понимать, что код специфичен для моей версии clang/llvm, операционной системы, времени года и все можен измениться.*

Все начинается со стандартного способа выдернуть типы аргументов/возвращаемных значений:

```cpp
template<class _Fp> class function; // undefined

template<class _Rp, class ..._ArgTypes>
class function<_Rp(_ArgTypes...)>
: public __function::__maybe_derive_from_unary_function<_Rp(_ArgTypes...)>,
  public __function::__maybe_derive_from_binary_function<_Rp(_ArgTypes...)>
```

Структуры ``__maybe_derive_from_unary_function`` и ``__maybe_derive_from_binary_function`` это просто ``std::unary_function`` и ``std::binary_function`` соответственно, если аргумент подходит, или пустая структура. Стоит отметить, что эти структуры устарели и лучше их не использовать.

Далее:
```cpp
{
    typedef __function::__base<_Rp(_ArgTypes...)> __base;
    typename aligned_storage<3*sizeof(void*)>::type __buf_;
    __base* __f_;
    ...
}
```

``__base`` это абстрактный класс-функтор. Он используется для ["type erasure"](http://www.cplusplus.com/articles/oz18T05o/). Грубо говоря, в шаблонном аргументе std::function указываются лишь требования к callable-объекту, но не его тип. Тип мы знаем в конструкторе/присвоении (``_Fp``):
```cpp
template<class _Fp> function(_Fp);
template<class _Fp> function& operator=(_Fp&&);
```

В конструкторе:
```cpp
template<class _Rp, class ..._ArgTypes>
template <class _Fp>
function<_Rp(_ArgTypes...)>::function(
    _Fp __f,
    typename enable_if
    <
        __callable<_Fp>::value &&
        !is_same<_Fp, function>::value
    >::type*)
    : __f_(0)
{
    if (__not_null(__f))
    {
        typedef __function::__func<
            _Fp,
            allocator<_Fp>, _Rp(_ArgTypes...)> _FF;
        if (sizeof(_FF) <= sizeof(__buf_) &&            // !!! [1]
            is_nothrow_copy_constructible<_Fp>::value)
        {
            __f_ = (__base*)&__buf_;
            ::new (__f_) _FF(std::move(__f));           // !!! [2]
        }
        else
        {
            typedef allocator<_FF> _Ap;
            _Ap __a;
            typedef __allocator_destructor<_Ap> _Dp;
            unique_ptr<__base, _Dp> __hold(             // !!! [3]
                __a.allocate(1),
                _Dp(__a, 1));
            ::new (__hold.get()) _FF(
                std::move(__f),
                allocator<_Fp>(__a));
            __f_ = __hold.release();
        }
    }
}
```

В контрукторе выясняется то, как хранится объект (пункты по [ меткам ]):
1. Если размер объекта меньше чем sizeof(void*) * 3 == 12, 24 байт
2. Создаем копию объекта ``__func`` внутри [std::aligned_storage](http://en.cppreference.com/w/cpp/types/aligned_storage)
3. Иначе создаем копию ``__func`` в хипе/аллокаторе через [std::unique_ptr](http://en.cppreference.com/w/cpp/memory/unique_ptr).

Сам ``__func`` - объект, который :
1. Наследуется от ``__base``, следовательно, реализует все абстрактные методы (в частности - вызов функтора).
2. Хранит в себе оригинальный callable-объект (``_Fp``)

```cpp
template<class _Fp, class _Alloc, class _Rp, class ..._ArgTypes>
class __func<_Fp, _Alloc, _Rp(_ArgTypes...)>
    : public  __base<_Rp(_ArgTypes...)>
{
    __compressed_pair<_Fp, _Alloc> __f_;
    ...
    explicit __func(_Fp&& __f, _Alloc&& __a)
        : __f_(piecewise_construct, std::forward_as_tuple(std::move(__f)),
                                    std::forward_as_tuple(std::move(__a))) {}
};
```

Самая интересная часть - как хранится callable-object - [__compressed_pair](http://en.cppreference.com/w/cpp/language/ebo)(привет, boost). Тоесть пихается функтор + аллокатор.
Вызывается:
```cpp
template<class _Fp, class _Alloc, class _Rp, class ..._ArgTypes>
_Rp
__func<_Fp, _Alloc, _Rp(_ArgTypes...)>::operator()(_ArgTypes&& ... __arg)
{
    typedef __invoke_void_return_wrapper<_Rp> _Invoker;
    return _Invoker::__call(__f_.first(), std::forward<_ArgTypes>(__arg)...);
}
```

### GCC version:
Тут все немного запутаннее:
* Создает и управляет логикой хранения/вызова функтора - _Base_manager.
* Данные непосредственно хранятся в _Any_data.
* И все это тонко намазанно на std::function.

В gcc реализации есть члены (недоступные для пользователя):
```cpp
static const std::size_t _M_max_size = sizeof(_Nocopy_types);
static const std::size_t _M_max_align = __alignof__(_Nocopy_types);
```
где ``_Nocopy_types``:
```cpp
class _Undefined_class;

union _Nocopy_types
{
    void*       _M_object;
    const void* _M_const_object;
    void        (*_M_function_pointer)();
    void        (_Undefined_class::*_M_member_pointer)();
};
```
Логично, что ``_M_max_size`` == 8, 16 байт (в понятиях Clang/LLVM == sizeof(void*) * 2), a ``_M_Const_object`` == 4, 8 (для 8 и 16 битной системы вы знаете значения сами).

Заметим, что в gcc нет базового объекта для функтора, тут есть ``union``:
```cpp
union [[gnu::may_alias]] _Any_data
{
    void*       _M_access()       { return &_M_pod_data[0]; }
    const void* _M_access() const { return &_M_pod_data[0]; }

    template<typename _Tp>
    _Tp&
    _M_access()
    { return *static_cast<_Tp*>(_M_access()); }

    template<typename _Tp>
    const _Tp&
    _M_access() const
    { return *static_cast<const _Tp*>(_M_access()); }

    _Nocopy_types _M_unused;
    char _M_pod_data[sizeof(_Nocopy_types)];
};
```
В ``_Any_data`` также хранится и буфер, для оптимизации мелких функторов.

Ну как же решается - хранить функтор динамически или нет? Посмотрим на ``_Base_manager``
```cpp
// _Base_manager:
    static const bool __stored_locally =
    (__is_location_invariant<_Functor>::value
     && sizeof(_Functor) <= _M_max_size
     && __alignof__(_Functor) <= _M_max_align
     && (_M_max_align % __alignof__(_Functor) == 0));

    typedef integral_constant<bool, __stored_locally> _Local_storage;

    static void
    _M_init_functor(_Any_data& __functor, _Functor&& __f)
    { _M_init_functor(__functor, std::move(__f), _Local_storage()); }

  private:
    static void
    _M_init_functor(_Any_data& __functor, _Functor&& __f, true_type)
    { ::new (__functor._M_access()) _Functor(std::move(__f)); }

    static void
    _M_init_functor(_Any_data& __functor, _Functor&& __f, false_type)
    { __functor._M_access<_Functor*>() = new _Functor(std::move(__f)); }

// function:
    template<typename _Res, typename... _ArgTypes>
    template<typename _Functor, typename, typename>
    function<_Res(_ArgTypes...)>::
    function(_Functor __f)
        : _Function_base()
    {
        typedef _Function_handler<_Res(_ArgTypes...), _Functor> _My_handler;

        if (_My_handler::_M_not_empty_function(__f))
        {
            _My_handler::_M_init_functor(_M_functor, std::move(__f));
            _M_invoker = &_My_handler::_M_invoke;
            _M_manager = &_My_handler::_M_manager;
        }
    }
```

Как мы видим, всегда дергается ``_M_init_functor(...)``, а в ней решается: создавать в куче или нет? За это отвечает переменная/тип ``__stored_locally``. В итоге, если объект меньше чем 8/16 байт (с учетом выравнивания) - ``std::function`` не будет выделять дополнительную память.

### Summary:
1. Каждый вызов std::function в Clang/LLVM - virtual-вызов. Это имеет свою цену ([ithare.com](http://ithare.com/wp-content/uploads/part101_infographics_v08.png)). В GCC такого нет (но от этого код стал немного запутанным).
2. Может аллоцировать память, а может и нет. Clang сделает это неохотнее, чем GCC.

К сожалению, аллокаторы в GCC не поддерживаются, а с std=c++17 перестают существовать вообще. Для Clang вы можете проверить сами, когда выделяется память. Интереснее всего - опыты с лямдами и результатом вызова std::bind т.к. эти объекты не стандартизированы.
Аллокаторы используются следующем образом:
```cpp
template <class T>
struct Allocator
{
    typedef T value_type;

    Allocator() = default;

    template <class U>
    constexpr Allocator(const Allocator<U>&) noexcept {}

    T* allocate(std::size_t n)
    {
        return (T*)std::malloc(n*sizeof(T));
    }

    void deallocate(T* p, std::size_t) noexcept
    {
        std::free(p);
    }
};

int main(void)
{
    Allocator<char> allocator;
    std::function<int()> v(std::allocator_arg, allocator, < any functor >);

    return 0;
}
```

Но есть несколько общеизвестных советов по использованию лямбд с ``std::function``:
1. sizeof([this] { ... }) = sizeof([&something] { ... }) <= sizeof([something] { ... })
2. В случае захвата по ссылке всего скоупа - размер лямбды ростет пропорционально количеству переменных в скоупе.
3. Иногда, чтобы избежать аллокацию памяти, лучше заменить ``std::bind`` на лябду с малым захватом.

*Так же в интернетах пишут, что gcc всегда выделяет память всегда для объектов-функторов, в моей версии (g++-7) это не так.*

### Links:
* [Small function optimization not applied to small objects](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=61909)
* [Deprecating Allocator Support in std::function](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0302r0.html)
* [Stackoverflow: how is stdfunction implemented](https://stackoverflow.com/questions/18453145/how-is-stdfunction-implemented)
