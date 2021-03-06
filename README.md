# Задание 3

Мобилизация.Гифки – сервис для поиска гифок в перерывах между занятиями.

Сервис написан с использованием [bem-components](https://ru.bem.info/platform/libs/bem-components/5.0.0/).

Работа избранного в оффлайне реализована с помощью технологии [Service Worker](https://developer.mozilla.org/ru/docs/Web/API/Service_Worker_API/Using_Service_Workers).

Для поиска изображений используется [API сервиса Giphy](https://github.com/Giphy/GiphyAPI).

В браузерах, не поддерживающих сервис-воркеры, приложение так же должно корректно работать, 
за исключением возможности работы в оффлайне.

## Структура проекта

  * `gifs.html` – точка входа
  * `assets` – статические файлы проекта
  * `vendor` –  статические файлы внешних библиотек
  * `service-worker.js` – скрипт сервис-воркера

Открывать `gifs.html` нужно с помощью локального веб-сервера – не как файл. 
Это можно сделать с помощью встроенного в WebStorm/Idea веб-сервера, с помощью простого сервера
из состава PHP или Python. Можно воспользоваться и любым другим способом.

# Решение

Изначально была прочитана рекомендованная статья про ServiceWorker. До начала выполнения задания я уже слышал про эту технология, но никогда с ней не работал. Воодушевленный новыми знаниями я приступил к поиску проблем в приложении. Однако был сильно удивлен и расстроен, когда все пошло совершенно не так как нужно. Главная проблема этого задания для меня оказалась достаточно простой. За годы проведенные в веб-разработке у меня выработался рефлекс "cmd + shift + R". А именно перезагрузка страницы без использования кэша. Как оказалось на ServiceWorker это тоже действует. Таким образом я потратил немало времени, чтобы понять, почему приложение не хочет загружаться в офлайне.

Проанализировав мою первую неудачу было принято решение - разобраться с особенностями дебага сервис воркеров. Оказалось, что инструменты разработчика chrome предосталвяют широкой набор инструментов для этого. На странице chrome://serviceworker-internals/ можно управлять любым сервис воркером, который использовался в браузере. Для дебага приложения с отключенным интернетом совершенно не обязательно выключать интернет. Достаточно воспользоваться стандартной галочкой "offline" в разделе Network инструментов разработчика chrome. На Service Worker она также прекрасно работает. Также стоит проверить галочку "disable cache" в разделе Network.

Теперь настроив Chrome правильным образом я вернулся к своим догадкам, которые сложились в процессе борьбы с непонятным поведением. Они оказались верными. Основная проблема, которой в статье уделяется очень много внимания, расположение файла service-worker.js. Необходимо перенести данный файл в корневую директорию, так как область видимости сервис воркера не распространяется на файлы которые лежат в директориях верхнего уровня. Соотвественно нужно найти все места где используется service-worker.js. Делаем это обычным поиском по проекту и заменяем на новый путь.

Следующая проблема, отстуствие в кэше точки входа в приложение, а именно gifs.html. Необходимо исправить функцию needStoreForOnline, а именно добавить следующую строчку:
```javascript
cacheKey.includes('gifs.html');
```
после этого файл gifs.html загружается из кэша

После проведенных исправлений появилась другая проблема - файлы стали загружаться только из кэша, даже при наличии интернет соединения. Для исправления ошибки было принято решение исследовать событие:
```javascript
self.addEventListener('fetch'...
```
В обработчике данного события стоит следующее условие:
```javascript
let response;
// Если needStoreForOnline true (файлы требует кэширования)
if (needStoreForOffline(cacheKey)) {
    // В следующией строке ошибка, так как файл всегда берется из кэша
    response = caches.match(cacheKey)
        .then(cacheResponse => cacheResponse || fetchAndPutToCache(cacheKey, event.request));
} else {
    response = fetchWithFallbackToCache(event.request);
}
```

Исправляем ошибочную строку следующим образом:

```javascript
response = fetchAndPutToCache(cacheKey, event.request);
```

Для кэширования файлов после первого запроса необходимо обратиться к обработчику

```javascript
self.addEventListener('install' ...
```
В цепочку промисов добавляем новую функцию следующим образом:

```javascript
const promise = preCacheAllFavorites()
        .then(() => preCacheAllFiles())  // Новая функция
        .then(() => self.skipWaiting())
        .then(() => console.log('[ServiceWorker] Installed!'));
```

И объявляем саму функцию:

``` javascript
// Положить в новый кеш все файлы
function preCacheAllFiles() {
    const files = [
        "./assets/blocks.js",
        "./assets/star.svg",
        "./assets/style.css",
        "./assets/templates.js",
        "./vendor/bem-components-dist-5.0.0/touch-phone/bem-components.dev.css",
        "./vendor/bem-components-dist-5.0.0/touch-phone/bem-components.dev.js",
        "./vendor/kv-keeper.js-1.0.4/kv-keeper.js",
        "./vendor/kv-keeper.js-1.0.4/kv-keeper.typedef.js",
        "./gifs.html",
        'https://yastatic.net/jquery/3.1.0/jquery.min.js'
    ];
    return caches.open(CACHE_VERSION)
        .then(cache => cache.addAll(files));
}
```

Для обновления файлов из статики можно изменить версию кэша (константа в коде) или удалить Service Worker используя chrome tools.


### Ответы

1. Данный метод позволяет перевести Service Worker в активный статус из статуса ожидания. Благодаря этому происходит запуск события activate.
2. Данный метод используется для того, чтобы установить данный сервис воркер в качестве активного сервис воркера в браузере клиента. Запуск данной функции происходит благодаря функции из первого пункта.
3. Для GET запросов с параметрами данный тип ключа будет работать неверно.
4. Данная цепочка вызовов используется для организации версионности кэша. Происходит проверка каждого файла на соответствующую версию. Если версия не совпадает - файл удаляется из кэша.
5. Клонирование запроса нужно потому, что запрос это поток, который можно прочитать только один раз. Поэтому необходим дополнительный экземпляр, который мы можем передать для записи в Service Worker.
