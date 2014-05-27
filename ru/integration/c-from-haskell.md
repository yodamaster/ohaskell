C из Haskell
------------

В ряде случаев нам необходима интеграция Haskell-кода с кодом, написанным на другом языке. Вот вам не выдуманный сценарий. Жила-была программная система, и было в ней несколько модулей. Один модуль решили переписать на Haskell, а вот другой, не менее важный модуль, переписать на Haskell нет времени/возможности/желания. Здесь и возникает потребность в том, чтобы подружить Haskell-код с кодом чужеродным.

Возможен и второй сценарий. Есть Haskell-приложение, и присутствует в нём одна секция, очень уж критичная к скорости выполнения. К счастью, имеется готовое решение, написанное на языке C. Опять-таки, без интеграции не обойтись.

Вы спросите, почему мы начинаем именно с языка C? Причина проста: стандарт гласит, что Haskell проще всего подружить со старым добрым C, что обусловлено, прежде всего, историческими причинами. Приступим.

### Hello foreign world!

Открываем `Main.hs` и пишем следующее:

```haskell
{-# LANGUAGE ForeignFunctionInterface #-}

module Main where

import Prelude hiding (sin)  -- Во избежание конфликта имён.
import Foreign.C             -- Наш проводник в мир языка C.

foreign import ccall sin :: CDouble -> CDouble

main :: IO ()
main = do
    putStrLn "Please input your number:"
    number <- getLine
    putStr "This is sinus: "
    print $ sin $ read number
```

Вывод будет таким:

```bash
Please input your number:
1
This is sinus: 0.8414709848078965
```

Вот и случилось маленькое чудо: мы вызвали стандартную функцию `sin` прямо из Haskell-приложения. Как же нам это удалось?

Прежде всего мы импортировали важный модуль:

```haskell
import Foreign.C
```

Этот стандартный модуль является нашим проводником в мир языка C. Кроме того, мы активизировали необходимое расширение Haskell:

```haskell
{-# LANGUAGE ForeignFunctionInterface #-}
```

А теперь рассмотрим следующую строку:

```haskell
foreign import ccall sin :: CDouble -> CDouble
```

Эта строка представляет для нас наибольший интерес, ведь главное чудо произошло именно здесь. С помощью ключевого слова `foreign` мы приглашаем чужака в наше Haskell-приложение. Далее следует слово import, потому что чужак импортный, ибо пришёл к нам извне. После него следует специальный стандартный идентификатор, показывающий тип чужака: `ccall` означает "вызов функции языка C". После идёт имя функции и её аргументы. Всё это можно прочесть так:

    foreign import      ccall      sin :: CDouble -> CDouble

    импортируем чужака: C-функция  sin    с сигнатурой как в Haskell

Обратите внимание на тип `CDouble`. Это - тип-обёртка. Объявление стандартной функции `sin` выглядит так:

```c
double sin(double x);
``` 

Однако компилятор Haskell не имеет ни малейшего представления о том, что такое `double`. Именно поэтому мы оборачиваем чужеродный тип `double` в специальный Haskell-тип `CDouble`, определённый в модуле Foreign.C.Types. После этого мы можем вызывать функцию `sin` так же, как если бы она была родной Haskell-функцией.

### Псевдоним

Поскольку в модуле `Prelude` тоже присутствует функция по имени `sin`, мы скрыли её, во избежание конфликта имён:

```haskell
import Prelude hiding (sin)  -- Не желаем видеть Haskell-функцию sin!
```

Однако можно пойти другим путём, а именно добавить псевдоним для чужака. В этом случае объявление изменится вот так:

```haskell
foreign import ccall "sin" c_sin :: CDouble -> CDouble
```

В кавычках - реальное имя чужака. После кавычек идёт псевдоним. Теперь вызов функции будет записан так:

```haskell
print $ c_sin $ read number
``` 

Кстати, если вдруг вы ошибётесь в написании реального имени чужой функции и напишете, например, так:

```haskell
foreign import ccall "sinu" c_sin :: CDouble -> CDouble
``` 

на стадии компоновки вы получите грозное сообщение:

```bash
Linking dist/build/Real/Real ...
Undefined symbols for architecture x86_64:
  "_sinu", referenced from:
      _s24m_info in Main.o
      _s2aR_info in Main.o
ld: symbol(s) not found for architecture x86_64
```

И ещё. Стандарт Haskell 2010 рекомендует указывать перед псевдонимом имя заголовочного файла, в котором объявлена функция-чужак. В нашем случае это будет выглядеть так:

```haskell
foreign import ccall "math.h sin" c_sin :: CDouble -> CDouble
```

Впрочем, признаюсь вам: имя заголовочника будет проигнорировано. Так что воспринимайте его просто как пояснительный комментарий о том, где живёт функция-чужак.

### Идём во внешний мир

Функция `sin` - чистая, поэтому с ней всё просто. Теперь попробуем пригласить в наш код чужака с побочными эффектами. Узнаем, который час:

```haskell
module Main where

import Foreign.C
import Foreign.Ptr  -- Для работы с C-указателями.

foreign import ccall time :: Ptr CTime -> IO CTime

main :: IO ()
main = do
    posixTime <- time nullPtr  -- Заменитель для NULL.
    print posixTime
```

В результате мы увидим старое доброе Unix-время:

```bash
1395350583
```

Теперь разберёмся. Во-первых, мы добавили дополнительный модуль `Foreign.Ptr`, необходимый для работы с C-указателями. Во-вторых, вспомним объявление стандартной функции time:

```c
time_t time(time_t* timer);
``` 

Чужак объявлен как принимающий значение типа `Ptr CTime`, то есть указатель на значение типа `CTime`, и возвращающий монаду `IO`, содержащую в себе значение типа `CTime` (ведь мы знаем, что получить реальное время можно лишь из внешнего мира). В итоге, передавая в качестве аргумента нулевой указатель, мы и получаем текущее время.

### А если void?

В функциях C часто используется тип `void`, однако в Haskell такого типа нет. Но поскольку `void` означает пустоту, создатели Haskell решили пойти самым простым путём: если функция принимает `void` (то есть не принимает ничего), в своей Haskell-сигнатуре она объявляется без аргумента.

Проверим это на стандартной функции clock, объявленной вот так:

```c
clock_t clock(void);
```

Пригласим её в наш код:

```haskell
foreign import ccall clock :: IO CClock  -- Аргумента нет.

main :: IO ()
main = do
    ticks <- clock
    print ticks
```

Но как же нам быть, если функця объявлена как _возвращающая_ тип `void`? Ведь мы не можем объявить Haskell-функцию без возвращаемого значения. Для таких случаев создатели Haskell предусмотрели маленький трюк: если C-функция не возвращает ничего, то в своей Haskell-сигнатуре она объявляется как возвращающая пустую `IO`-монаду.

Например, стандартная функция `exit` объявлена так:

```c
void exit(int status);
```

Пригласим её в гости:

```haskell
foreign import ccall exit :: CInt -> IO ()  -- Возвращает... ничего.

main :: IO ()
main = exit 2  -- Выходим из приложения со статусом 2.
```

При запуске получим:

```bash
shell returned 2
```

Кстати, тип `void*`, часто используемый в языке C, соответствует типу `Ptr ()`, то есть указатель на ничего.

### Структура и память

Следующий пример поинтереснее. Поработаем с локальным временем с помощью стандартных функций `time`, `localtime` и `asctime`:

```c
int main() {
    time_t rawtime;
    struct tm* timeinfo;

    time( &rawtime );
    timeinfo = localtime( &rawtime );

    printf( "%s", asctime( timeinfo ) );
}
```

Результат:
 
```bash
Tue Mar 25 08:33:02 2014
```

И так нам приглянулись эти функции, что решили мы использовать их в нашем Haskell-приложении. Сказано - сделано. Для начала вспомним родные объявления этих функций, чтобы написать их Haskell-аналоги:
 
```c
time_t time(time_t* timer);

struct tm* localtime(const time_t* timer);

char* asctime(const struct tm* timeptr);
```

С функцией `time` мы уже знакомы, а вот две её коллеги заслуживают нашего пристального внимания. Функция `localtime` возвращает указатель на структуру `tm`. А функция `asctime`, во-первых, принимает этот указатель в качестве единственного аргумента, а во-вторых, возвращает старую добрую C-строку.

Главный вопрос: что нам делать со структурой? Ведь тип `struct tm`, хотя и определён в стандартном заголовочнике `<time.h>`, не является стандартным типом языка C, поэтому ему нет аналога в модуле `Foreign.C.Types`. А это значит, нам потребуется небольшой трюк. Пишем:

```haskell
module Main where 

import Foreign.C
import Foreign.Ptr
import Foreign.Marshal.Alloc

data CTmStruct = CTmStruct         -- Наш аналог для типа struct tm.
type CTmStructPtr = Ptr CTmStruct  -- Определяем указатель на этот аналог.

foreign import ccall time :: Ptr CTime -> IO CTime
foreign import ccall localtime :: Ptr CTime -> IO CTmStructPtr
foreign import ccall asctime :: CTmStructPtr -> CString
```

Трюк оказался предельно простым: мы ввели новый тип `CTmStruct`, "заменяющий" C-тип struct tm. Но постойте, ведь тип `CTmStruct` не более чем пустышка, как же он может заменить тип `struct tm`? А он и не должен его заменять, он должен лишь _отобразить_ его. Мы говорим: "Теперь у нас есть тип `CTmStruct`, и есть указатель на него, но что этот тип представляет собой на самом деле мы не знаем и знать не хотим". Именно поэтому импорт функции `localtime` выглядит так:

```haskell
foreign import ccall localtime :: Ptr CTime -> IO CTmStructPtr
```

Да, мы знаем, что эта функция принесёт из внешнего мира значение типа  `CTmStruct`, но поскольку мы не будем копаться во внутренностях этого значения, нам и незачем знать, что оно собою представляет. Мы даже могли бы обойтись без типа `CTmStruct` и воспользоваться "пустым указателем", написав просто:

```haskell
type CTmStructPtr = Ptr ()  -- Просто указатель на нечто.
```

С этим разобрались. Но есть ещё одна трудность. Вспомним код на C:

```c
    time_t rawtime;
    struct tm* timeinfo;

    time( &rawtime );                  -- Взятие адреса переменной?    
    timeinfo = localtime( &rawtime );  -- Опять берём адрес??
```

Здесь нужно совершить действие, обычное для языка C: применить к некоторому значению унарный оператор взятия адреса `&`. Однако в Haskell такого оператора нет. Следовательно, нам не обойтись без ещё одного трюка.

Определим функцию, работающую с нашими друзьями-чужаками:

```haskell
timeObtainer :: Ptr CTime -> IO String
timeObtainer timePtr = do
    time timePtr
    tm <- localtime timePtr
    peekCString $ asctime tm
```

Всё просто: вызываем три функции так же, как и в том примере на C. Даже новая для нас функция `peekCString` предельно ясна, ибо превращает C-строку в Haskell-строку. В общем, ничего примечательного в `timeObtainer` нет. Но, глядя на эту функцию, у вас обязательно возникнет вопрос о том, откуда же мы возьмём этот `timePtr`? Тут-то и выходит на сцену упомянутый выше трюк. Определим ещё одну функцию:

```haskell
getTime :: IO String
getTime = alloca $ timeObtainer  -- Трюк со взятием адреса.
```

Мы видим новую для нас функцию `alloca`, определённую в стандартном модуле `Foreign.Marshal.Alloc`. Вот её объявление:

```haskell
alloca :: Storable a => (Ptr a -> IO b) -> IO b
```

Именно она предоставляет указатель для нашей функции `timeObtainer`. Обратите внимание: тип `a`, указатель на который принимает функция `timeObtainer`, должен входить в контекст класса `Storable`. Суть этого класса ясна из его названия: значение типа, относящегося к этому классу, можно сохранить в "сырой" памяти. Таким образом, функция `timeObtainer` имеет дело с указателем `timePtr`, _уже_ инициализированным реальным адресом локального значения типа `CTime`, и всё благодаря функции `alloca`.

Теперь пишем:

```haskell
main :: IO ()
main = do
    timeFromCWorld <- getTime
    putStr timeFromCWorld
```

и получаем ожидаемый результат:

```bash
Wed Mar 26 23:42:15 2014
```

### Работаем с внутренностями

Работая с указателем на значение типа `CTmStruct`, мы не проявляли ни малейшего интереса к содержимому этого значения. Именно поэтому мы смогли полностью абстрагироваться от внутренностей `CTmStruct` и написать так:

```haskell
type CTmStructPtr = Ptr ()  -- Просто указатель на нечто.
```

Однако в ряде случаев мы _обязаны_ работать с содержимым чужеродного значения. Канонический пример: необходимая нам C-функция ожидает на вход указатель на структуру, поля которой нам необходимо выставить в некие известные нам значения.

Рассмотрим функцию `utime`. Эта функция, определённая в стандартном заголовочнике `<utime.h>`, предназначена для установки времени последнего доступа к файлу и времени его последней модификации. Вот как это выглядит в C:

```c
#include <utime.h>
#include <stdio.h>

int main() {
    struct utimbuf real_time;
    real_time.actime = 1396126594;  /* Дату мы как-то получили... */
    real_time.modtime = 1396126594; /* И эту тоже. */

    int code = utime( "/Users/dshevchenko/test.c", &real_time );
    if( code == 0 ) {
        printf( "It's done." );
    } else {
        printf( "Something wrong..." );
    }
}
```

И вот случилось так, что нам позарез понадобилась функция `utime` в нашем Haskell-приложении. В этом случае абстрактным указателем мы уже не обойдёмся, ведь нам придётся полноценно работать со значением типа `struct utimbuf`. Как же это делается?

Для начала создадим новый файл `UTime.hsc` в каталоге `src/Utils`. Вы заметили? У этого файла какое-то странное расширение, `.hsc` вместо привычного `.hs`. А всё потому, что этот файл является своего рода посредником между Haskell и C: содержащийся в нём код представляет собой необычную смесь из C-кода и Haskell-кода. Откроем этот файл и напишем следующее:

```haskell
module Utils.UTime where

#include <utime.h>  -- Знакомимся с C-типом struct utimbuf...

import Foreign.C
import Foreign.Ptr
import Foreign.Storable

-- Тип-посредник с двумя такими же полями, как в структуре utimbuf.

data UTimeBuffer = UTimeBuffer { accessTime :: CTime
                               , modificationTime :: CTime
                               }

type UTimeBufferPtr = Ptr UTimeBuffer

instance Storable UTimeBuffer where
    sizeOf _ = (#const sizeof (struct utimbuf))
    alignment _ = alignment (undefined :: CInt)

    peek ptr = do
        c_actime <- (#peek struct utimbuf, actime) ptr
        c_modtime <- (#peek struct utimbuf, modtime) ptr
        return $ UTimeBuffer c_actime c_modtime

    poke ptr realValue = do
        (#poke struct utimbuf, actime) ptr (accessTime realValue)
        (#poke struct utimbuf, modtime) ptr (modificationTime realValue)

-- Наше приглашение для функции-чужака.  
foreign import ccall "utime.h utime"
                     c_utime :: CString -> UTimeBufferPtr -> IO CInt
```

Необычно выглядит, не правда ли? Давайте разбираться.

Первое новшество - тип `UTimeBuffer`. Поскольку мы не можем работать с нестандартным C-типом напрямую, нам необходим Haskell-посредник. В этом типе определены два поля тех же типов, что и поля в структуре `utimbuf`.

Второе новшество - экземпляр класса `Storable` для нашего типа `UTimeBuffer`. Как уже было упомянуто выше, мы обязаны добавить наш тип в контекст этого класса, если планируем сохранять значения нашего типа в "сырой" памяти. Здесь определяются четыре метода, заслушивающие нашего внимания.

Метод `sizeOf` возвращает размер нашего типа, подобно ключевому слову `sizeof` в языке C. Метод `alignment` возвращает величину [выравнивания структуры](https://en.wikipedia.org/wiki/Data_structure_alignment). Метод `peek` читает значение, расположенное по указанному адресу. Ну а метод `poke`, напротив, записывает некое значение по указанному адресу.

Вы спросите, что это за странный синтаксис? Какие-то круглые скобки и решётки... А это именно то, о чём было сказано выше: содержимое `.hsc`-файлов не является полноценным кодом на Haskell. Этот файл - препроцессорный, и чтобы превратить его в настоящий Haskell-код, нужен препроцессор под названием [`hsc2hs`](http://www.haskell.org/ghc/docs/7.6.3/html/users_guide/hsc2hs.html). Однако непосредственное его использование вам едва ли понадобится, ведь наша подруга `cabal` сама разберётся, какие файлы проекта являются препроцессорными, и сама выполнит для них команду `hsc2hs`. Поэтому просто добавьте имя модуля `Utils.UTime` в `Real.cabal`, в параметре `other-modules`.

Теперь рассмотрим эти препроцессорные инструкции внимательнее. Вот `sizeOf`:

```haskell
sizeOf _ = (#const sizeof (struct utimbuf))
```

Мы говорим: "Метод `sizeOf`, применяемый к некоторому значению нашего типа, вернёт целочисленную константу, соответствующую размеру типа `struct utimbuf`". Обратите внимание: мы используем полноценное выражение на языке C:

```haskell
sizeof (struct utimbuf)
```

включающее в себя родное имя той самой структуры, с которой мы собираемся работать. И чтобы препроцессор знал, о какой структуре идёт речь, мы включаем заголовочный файл, в котором она определена:

```haskell
#include <utime.h>
```

А вот метод `peek`:

```haskell
peek ptr = do
    c_actime <- (#peek struct utimbuf, actime) ptr
    c_modtime <- (#peek struct utimbuf, modtime) ptr
    return $ UTimeBuffer c_actime c_modtime
```

Этот метод должен вернуть Haskell-значение, находящееся по адресу `ptr`, а значит, мы последовательно считываем два значения, входящие в нашу структуру-посредника. Рассмотрим первое прочтение:

```haskell
c_actime <- (#peek struct utimbuf, actime) ptr
```

Мы говорим: "Читаем значение типа `struct utimbuf`, находящееся по адресу `ptr`, и значение его поля `actime` ассоциируем с идентификатором `c_actime`". После прочтения обоих полей мы создаём значение нашего типа-посредника `UTimeBuffer` и возвращаем его в `IO`-монаду.

Как видите, данный препроцессорный код читается довольно легко, ведь по своему виду он очень приближен к родному Haskell-коду. Теперь откроем наш `Main.hs` и напишем в нём следующее:

```haskell
module Main where

import Utils.UTime
import Foreign.C
import Foreign.Marshal.Alloc
import Foreign.Storable

main :: IO ()
main = do
    path <- newCString "/Users/dshevchenko/test.c"
    utimeBufferPtr <- malloc
    poke utimeBufferPtr realTime
    code <- c_utime path utimeBufferPtr
    putStrLn $ if code == 0 then "It's done" else "Something wrong..."
    where realTime = UTimeBuffer { accessTime = 1396123253
                                 , modificationTime = 1333050979
                                 }
```

Наше внимание привлекают следующие строки:

```haskell
    path <- newCString "/Users/dshevchenko/test.c"
    utimeBufferPtr <- malloc    -- Повеяло языком C, не правда ли?
    poke utimeBufferPtr realTime
```

В первой мы используем функцию `newCString`, превращающую Haskell-строку в C-строку. Далее мы выделяем память для нашей структуры-посредника с помощью стандартной функции `malloc`. А потом мы сохраняем значение `realTime` по адресу, ассоциированному с указателем `utimeBufferPtr`.

Обратите внимание и на нашего старого знакомого - на составной тип, формируемый с помощью фигурных скобок:

```haskell
    where realTime = UTimeBuffer { accessTime = 1396123253
                                 , modificationTime = 1333050979
                                 }
```

Не важно, откуда мы получили эти posixtime-значения, важно, что когда мы запустим этот код, получим ожидаемый результат:

```bash
It's done
``` 

Поверьте мне на слово - это работает, и время модификации файла действительно изменяется.

### А убираться кто будет?!

Ай-яй-яй, мы чуть не забыли... Вот наш код:

```haskell
    path <- newCString "/Users/dshevchenko/test.c"
    utimeBufferPtr <- malloc  -- Радостно выделяем память в куче...   
    poke utimeBufferPtr realTime
    code <- c_utime path utimeBufferPtr
    putStrLn $ if code == 0 then "It's done" else "Something wrong..."
    ...
    -- А освободить-то забыли...  
```

В самом деле, выделить-то мы выделили, а убираться кто будет? Встроенный в Haskell уборщик мусора здесь нам не помощник, ведь взаимодействие с чужаками вынудило нас выделить память вручную, как в языке C. А если выделили явно - нужно явно и освободить.

А кстати, вдруг я вас обманываю? Действительно ли утечка имеет место в нашем примере? Обратимся к нашему верному помощнику [valgrind](http://valgrind.org). Выполним:

```bash
$ valgrind --leak-check=full ./dist/build/Real/Real
```

И вот грустный, но вполне ожидаемый отчёт:

```bash
==25303== 16 bytes in 1 blocks are definitely lost in loss record 4 of 14
==25303==    at 0xE0D6: malloc (vg_replace_malloc.c:274)
==25303==    by 0x1000019DA: s25W_info (in ./dist/build/Real/Real)
```

Утекло. А чтобы этого избежать, воспользуемся стандартной функцией free и напишем так:

```haskell
...
    path <- newCString "/Users/dshevchenko/test.c"
    utimeBufferPtr <- malloc  -- Радостно выделяем память в куче...   
    poke utimeBufferPtr realTime
    code <- c_utime path utimeBufferPtr
    free utimeBufferPtr       -- И столь же радостно освобождаем.
    putStrLn $ if code == 0 then "It's done" else "Something wrong..."
    ...
```

Теперь проверим:

```bash
$ valgrind --leak-check=full ./dist/build/Real/Real
```

И вот он, радостный отчёт:
 
```bash
==25356==  LEAK SUMMARY:
==25356==    definitely lost: 0 bytes in 0 blocks
```

Память освобождается.

### В сухом остатке

* Haskell легко подружить с языком C.
* Чтобы вызывать C-функцию из Haskell, необходимо объявить Haskell-прототип этой функции.
* Все стандартные C-типы имеют соответствующие Haskell-обёртки.
* Если C-функция принимает `void`, её Haskell-прототип не имеет аргументов. Если C-функция возвращает `void`, её Haskell-прототип возвращает `IO ()`.
* Для взятия адреса локальной переменной используется функция `alloca`.
* Для работы с C-структурой следует создать посреднический Haskell-тип.
* Экземпляр класса `Storable` для посреднического типа пишется на препроцессорном коде в `.hsc`-файле.
* При ручном выделении памяти её необходимо вручную и удалить.

### Пробуем

Код из этой главы доступен онлайн.

<span><a href="https://www.fpcomplete.com/ide?title=c-from-haskell&paste=https://raw.githubusercontent.com/denisshevchenko/ohaskell-code/master/code/integration/c-from-haskell/Main.hs" class="fpcomplete_code" target="_blank">Открыть в FP IDE</a></span>
<span class="buttons_space"></span>
<span><a href="https://github.com/denisshevchenko/ohaskell-code/blob/master/code/integration/c-from-haskell/Main.hs" class="github_code" target="_blank">Открыть на GitHub</a></span>
