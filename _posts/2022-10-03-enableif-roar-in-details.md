## std::enable_if або як переконати всіх перейти на С++ 20

В попередньому тексті був описаний SFINAE, тут хотів би детальніше розглянути його застосування та найбільш відомий, в певних колах, `std::enable_if`.
Далі невеличке введення в С++ 20 concepts. 


### std::enable_if 
Отже, [cppreference](https://en.cppreference.com/w/cpp/types/enable_if) каже: 

> Ця метафункція є зручним способом використання SFINAE, до приходу концепцій C++20

`std::enable_if` можна використовувати в кількох формах, зокрема:
* як додатковий аргумент функції
* як тип повернення
* як шаблон класу або параметр шаблону функції

Дуже проста релізація наведена на тому ж таки  [cppreference](https://en.cppreference.com/w/cpp/types/enable_if): 
```cpp

template<bool B, class T = void>
struct enable_if {};
 
template<class T>
struct enable_if<true, T> { typedef T type; };

template<bool _Cond, typename _Tp = void>
using enable_if_t = typename enable_if<_Cond, _Tp>::type;
```

### Перевантаження через std::enable_if
За допомогою `std::enable_if` можна дозволяти виклик певної функції лише вибраних типів. Наприклад:

```cpp
template<class T>
typename std::enable_if<(sizeof(T) > 2)>::type pack2bytes(T& buffer)
{
    buffer = 0xBEAF;
}
```

Проте даний варіант має недоліки - тип повернення. Читати його доволі важко і ми не можемо власне його використати для іншиї цілей. 
Поширенішою практикою є застосування `std::enable_if` як параметр шаблону функції: 

```cpp
template<typename T, typename = std::enable_if_t<(sizeof(T) > 2)>>
void pack2bytes(T& buffer)
{
    buffer = 0xBEAF;
}
```
або ж навіть читабельніше:
```cpp
template<typename T>
using EnableIfSizeGreater2 = std::enable_if_t<(sizeof(T) > 2)>;

template<typename T, typename = EnableIfSizeGreater2>>
void pack2bytes(T& buffer)
{
    buffer = 0xBEAF;
}
```
Також доволі практичний приклад при використанні разом з конструкторами та іншими метафункціями:

```cpp
class Person
{
public:
   template<class T, typename = std::enable_if_t<
                                std::is_convertible_v<T, std::string>>>
   Person(T&& name) : m_name(std::forward<T>(name)) {}

private:
    std::string m_name;
};
```
### Приклади 

Далі просто наведу кілька прикладів з реальних проектів (їх сотні, просте наведемо лиш ті що підпалися під руку). 

[Qt:corelib/kernel/qmath.h](https://github.com/qt/qtbase/blob/v6.3.2/src/corelib/kernel/qmath.h#L311): 
```cpp
template <typename T, std::enable_if_t<std::is_integral_v<T>, bool> = true>
constexpr inline double qDegreesToRadians(T degrees)
{
    return qDegreesToRadians(static_cast<double>(degrees));
}
```
[Qt:src/corelib/tools/qhash.h](https://github.com/qt/qtbase/blob/dev/src/corelib/tools/qhash.h#L33):
```cpp
template <typename T>
constexpr inline bool HasQHashOverload<T, std::enable_if_t<
    std::is_convertible_v<decltype(qHash(std::declval<const T &>(), std::declval<size_t>())), size_t>
>> = true;
```
[clang:lib/AST/Interp/Integral.h](https://github.com/llvm/llvm-project/blob/main/clang/lib/AST/Interp/Integral.h#L168)
```cpp
template <unsigned SrcBits, bool SrcSign>
static std::enable_if_t<SrcBits != 0, Integral>
from(Integral<SrcBits, SrcSign> Value) {
  return Integral(Value.V);
}
```

[swift:include/swift/Basic/StableHasher.h](https://github.com/apple/swift/blob/main/include/swift/Basic/StableHasher.h#L145):
```cpp
  template <
      typename T,
      typename std::enable_if<std::is_integral<T>::value>::type * = nullptr>
  void combine(T bits) {
    constexpr auto endian = llvm::support::endianness::little;
    uint8_t buf[sizeof(T)] = {0};
    bits = llvm::support::endian::byte_swap<T>(bits, endian);
    std::memcpy(buf, &bits, sizeof(T));
    combine<sizeof(T)>(buf);
  }
```

Як бачимо, на практиці воно не дуже читабельно. Код змушує видивлятися певні регіони в тій чи іншій області, перш ніж зрозуміти що до чого. 

І тут на порятунок мають прийти концепти. 

### C++20 та концепти
Концепти покликані спростити життя С++ програмістів (як і все в С++). Ідея доволі проста - це описати правила, яким має підкорятися той чи інший тип переданий як вхідний. 
Наприклад правило, яка накладається на тип, можна описати концептом: 
```cpp
template<typename T>
concept GreaterThan2 = sizeof(T) > 2;

template<typename T>
concept LessThan6 = sizeof(T) < 6;
```

Тут використовується нове для С++ ключове слово `concept`. Далі є кілька варіантів використання концепту (на прикладі тої ж таки `pack2bytes`): 
```cpp 
template<GreaterThan2 T>
void pack2bytes(T& buffer)
{
    buffer = 0xBEAF;
}
```
або за допомогою`requires`:
```cpp
template<class T> requires GreaterThan2<T> && LessThan6<T>
void pack2bytes(T& buffer)
{
    buffer = 0xBEAF;
}
```
От така історія. Висновки робіть самі. Далі буде... 


