## Свиня є... або як зрозуміти SFINAE
Дуже часто світ, в якому живуть С++ програмісти, поділений на кілька окремих островів між якими океани невизначеності, суперечок та таємниць. Один з таких островів стоїть на шаблонах, як тектонічній плиті, що рухається і рухається все з новими версіями С++.
Саме тут виникає таке поняття, як SFINAE.
 
> Не всі можуть дивитися в SFINAE, вірніше дивитися можуть всі, але зрозуміти лиш деякі (с).
 
Детальніше про нього спробую описати та пояснити тут.
 
### То свиня є чи немає?
 
SFINAE ("Substitution Failure Is Not An Error", укр. «Помилка підстановки не є помилкою»,  "cвиняЄ") - виключення з правил, яке дозволяє ігнорувати ті чи інші перевантаження функції (помилку підстановки) при генерації потрібної. Це виключення допомагає уникнути генерування конструкцій, які не мають сенсу (як ліберали в рашці). SFINAE виникло, коли одні правила мови суперечили іншим.
 
### Давайте детальніше...
 
> Коли компілятор бачить виклик перевантаженої функції, він повинен розглядати кожну з них окремо. Оцінювати аргументи виклику та вибрати кандидата, який найкраще відповідає критеріям.
> У випадках, коли набір кандидатів для виклику включає шаблони функцій, компілятор спочатку повинен визначити, які аргументи шаблону слід використовувати для цього кандидата, а потім замінити ці аргументи у списку параметрів функції та в її типі повернення(виконати підстановку, анг. substitution), а потім ще й оцінити, наскільки добре вона збігається (так само, як і звичайна функція). Однак процес заміни може зіткнутися з проблемами: ці підстановки можуть генерувати конструкції, які завідомо грають не по правилах і провокують помилки, які роблять шаблонне програмування дуже і дуже не зручним (куди вже далі...)
>
> C++ Templates. The Complete Guide by David Vandevoorde, Nicolai M. Josuttis, Douglas Gregor
 
### Тексту багато, глянемо код.
 
Подивимося на приклад з тої ж таки книги `C++ Templates. The Complete Guide`.
 
```cpp
#include <vector>
 
template<typename T, unsigned N>
T* begin(T (&array)[N])
{
   return array;
}
 
template<typename Container>
typename Container::iterator begin(Container& c)
{
   return c.begin();
}
int main()
{
   std::vector<int> v;
   int a[10];
   ::begin(v);
   ::begin(a);
}
```
 
Ми маємо дві шаблонні функції `begin`. Якби не існувало SFINAE, компілятор створив би 4 функції, оскільки ми викликаємо їх з двома різними типами в `main`.
 
Це б призвело до помилки компіляції, через те що `begin` генерувався б для обох типів агрументів, в обох функціях ще на етапі пошуку найкращого кандидата.
 
Перша підстановка `T = std::vector<int>` призвела б до помилки в першій функції з аргументом масивом: 
```
error: no matching function for call to ‘begin(std::vector<int>&)’
note: mismatched types ‘T [N]’ and ‘std::vector<int>’
```
 
Друга підстановка `T = int[10]`, в другій функції з аргументом вектором, призвела б до помилки при її виклику з масивом:
 
```
error: ‘iterator’ is not a member of ‘int’
```
 
Але, на наше щастя, **помилка підстановки не є помилкою**, тому компілятор успішно проігнорував завідомо "невірний" код і знайшов для нас вихід - лише дві потрібні нам функції:
 
```cpp
template<>
int * begin<int, 10>(int (&array)[10])
{
   return array;
}
 
template<>
std::vector<int>::iterator begin<std::vector<int>>(std::vector<int> & c)
{
   return c.begin();
}
```
 
З цього моменту принцип SFINAE (далі: "cвиняЄ") мав би бути украденим у Вашій голові та перехід на темну сторону сили повільно почався.
 
### Як "свиняЄ" працює на практиці?
 
Колись до мене прийшов мій старий друг Бьярне і сказав: "Шаблони це наше майбутнє". Я промовчав, але хотів було заперечити, та він швидко вибіг і повернувся Данію. Я згадав Подерв'янського.
Виявляється багато чого корисного можна зробити маючи таке цікаве виключення з правил.
Перше що приходить в голову пересічному експерту по "свинях" це `type_traits`. Бувалі згадають бібліотеки `Loki` та `boost::hana` та безліч інших.
 
Почнемо з `type_traits`. Більшість корисних текстів вже написана до нас і для нас. Майже всі (якщо не всі) сутності тут працюють по принципу "свиняЄ" або залежні від нього.
 
Аби детальніше розглянути це на практиці, напишемо спрощену версію `std::is_convertible`.
 
```cpp
template<typename FROM, typename TO>
struct IsConvertibleHelper {
private:
   static void aux(TO);
 
   template<typename F, typename T,
            typename = decltype(aux(std::declval<F>()))>
   static std::true_type test(void*);
 
   template<typename, typename>
   static std::false_type test(...);
 
public:
   using Type = decltype(test<FROM, TO>(nullptr));
};
 
template <typename FROM, typename TO>
struct IsConvertibleT : IsConvertibleHelper<FROM, TO>::Type {};
 
template<typename FROM, typename TO>
constexpr bool isConvertible = IsConvertibleT<FROM, TO>::value;
 
 
int main()
{
   if constexpr (isConvertible<int, int>) {
       std::cout << "<int, int>";
   }
 
   if constexpr (isConvertible<const char*, std::string>) {
       std::cout << "<std::string, const char*>";
   }
 
   if constexpr (isConvertible<double, std::string>) {
       std::cout << "<double, std::string>";
   }
 
   return 0;
}
```
Ідея тут проста. Аби перевірити чи можна створити один тип з іншого, ми маємо функцію `aux` яка приймає лише `TO`. Код спробує "викликати" цю функцію з `FROM`. Це основа нашого "свиняЄ", саме по цьому критерію ми будемо визначати наш результат. Варто відмітити що "викликів" функцій як таких не існує, натомість компіляція передбачає процес перевірки типів в тих чи інших обставинах.
 
Тобто, ми хочемо отримати наступне:
 
```
std::true_type test(void*) // для потрібних типів
std::false_type test(...) // для тих, які несумісні
```
 
Перевівши це на простий не шаблонний приклад, можна описати процес виклику функції з параметром, який автоматично переводиться в потрібний:
```cpp
static void aux(int) {}
int main()
{
   aux(long(1)); // long переводиться в int
   aux(short(1)); // short переводиться в int
   aux(std::string()); // помилка: std::string не можна перевести в int
}
```
 
На щастя, з шаблонами все цікавіше. Закон шаблонів в С++ дуже простий: там де є один, буде й інший.
 
Отже, повернемося до `IsConvertibleHelper`. "свиняЄ" функція виглядає доволі цікаво
```cpp
template<typename F, typename T,
        typename = decltype(aux(std::declval<F>()))>
   static std::true_type test(void*);
```
По перше, це шаблон в шаблоні. Нам треба відокремити контекст в якому викликався `TO` та `FROM` на вищому рівні, і оминути помилку
компіляції в разі неспівпадіння типів, скажемо при `isConvertible<double, char*>`. Це робиться таким прийомом. 
 
Далі увагу звернемо на `typename = decltype(aux(std::declval<F>()))>`. `std::declval` в комбінації з `decltype` це дуже поширене явище.
Тут ми пробуємо "викликати" `aux` з типом `F = FROM`.
Отже, якщо конвертація `FROM->TO` можлива, компілятор генерує саме функцію `std::true_type test(void*)`. `std::true_type` тут використаний задля спрощення. Ми можемо повертати будь який тип, проте кінцева перевірка буде важчою.
 
Відокремити "аудиторію" типів ми змогли. Що дати для всіх решти? Щось, що буде вертати `std::false_type`. І тут допомагає оператор `...` (ellipsis), як мітка "найгіршого кандидата", використовуємо [variadic argument](https://en.cppreference.com/w/cpp/language/variadic_arguments)
 
```cpp
template<typename, typename>
static std::false_type test(...);
```
 
От і все. Давайте спробуємо описати що генерує компілятор для `isConvertibel<int, long>`
 
```cpp
template<>
struct IsConvertibleHelper<int, long>
{

private:
    static void aux(long);

    template<typename F, typename T, typename = aux(std::declval<F>())>
    static std::true_type test(void *);

    template<>
    static std::true_type test<int, long, void>(void *);

    template<typename, typename>
    static std::false_type test(, ...);

    template<>
    static std::false_type test<int, long>(, ...);


public:
    using Type = std::true_type;
};

int main()
{
    isConvertible<long, int>;
}
```
 
Існує безліч інших аспектів використання "свиняЄ" принципу, та описати їх напевно не вистачить часу. Тому рекомендую прочитати вже згадану книгу `C++ Templates. The Complete Guide` та глянути `type_traits` власними очима. Наразі дуже багато бібліотек використовують "свиняЄ" на практиці (stl, boost, Qt, asio, і тд). Читання їх коду теж допоможе розібратися як з описаним принципом, так і з іншими аспектами мови.
 
Подальшим розвитком будемо вважати `std::enable_if` та концепти, які прийшли з C++ 20. Спробуєм детально описати їх в наступних текстах.
 
Дякую за увагу!

#cpp #cxx #sfinae #ukr
 
