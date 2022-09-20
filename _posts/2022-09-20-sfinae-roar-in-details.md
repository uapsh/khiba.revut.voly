## Тут буде щось ...

```cpp
template<class T, typename = std::enable_if<std::has_content_v<T>>
void post(T content)
{
    content.get();
}
```
