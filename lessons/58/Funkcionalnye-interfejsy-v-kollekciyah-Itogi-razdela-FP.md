![](../../commonmedia/header.png)

***

   

Функциональные интерфейсы в коллекциях. Итоги раздела ФП
========================================================

Сегодняшний урок будет заключительным в разделе «Функциональное программирование в Java». И его мы разделим на две части.

Первая будет посвящена знакомству с новыми (и не очень) методами коллекций, принимающими лямбда-выражения в качестве параметров.

Почему сейчас? Преимущественно, потому что раньше мы не были знакомы с лямбда-выражениями, а когда познакомились – открылись новые, более приоритетные темы, в контексте ФП.  
Какие-то из методов, которые мы рассмотрим сегодня, уже встречались нам ранее, какие-то – нет. Какие-то можно рассматривать как экономию на использовании Stream API (по крайней мере, самых простых и коротких цепочек), какие-то не имеют прямых аналогий, но просто полезны в рамках своих задач.

Вторая часть урока будет посвящена тому, какие еще проявления ФП существуют в Java. Она будет ознакомительная и призвана показать, что ФП – это не только Stream API. Возможно, именно такое мнение сложилось у вас сейчас (или сложится чуть позже).

Итак, начинаем.

### Функциональный интерфейсы в коллекциях

###  Iterable

#### void forEach(Consumer<T> action)\*

> \*Здесь и ниже, в параметризованных методах опущены ограниченные вайлдкарды. Это не мешает восприятию параметров, но делает заголовки более читабельными. Иными словами, в рамках статьи:  
> **<? extends T> -> <T>**,  
> **<? super T> -> <T>**.

С данным методом мы уже знакомы. Является заменой цикла _foreach_. Принимает параметром лямбда-выражение, которое будет выполнено для каждого элемента коллекции.

### Collection

#### T\[\] toArray(IntFunction<T\[\]> generator)

Аналогичен схожему методу в _Stream_. По лямбда-выражению, переданному параметром, создает массив нужного типа, содержащий все элементы коллекции.

#### boolean removeIf(Predicate<E> filter)

Один из действительно новых для нас методов. Удаляет все элементы в коллекции, для которых лямбда-выражение, переданное параметром, возвращает _true_. Таким образом, предлагает намного более гибкий и удобный функционал, чем классический _remove()_.

**Пример**. Удалим из коллекции целых чисел все четные:

```java
var list = new ArrayList<>(List.of(1, 2, 3, 4));
list.removeIf(i -> i % 2 == 0);
list.forEach(System.out::print); //13
```

  

### List

#### void replaceAll(UnaryOperator<E> operator)

Заменяет все элементы списка в соответствии с переданной лямбдой.

**Пример 1**. Увеличим все элементы списка целых чисел на единицу:

```java
var list = new ArrayList<>(List.of(1, 2, 3, 4));
list.replaceAll(i -> ++i);
list.forEach(System.out::print); //2345
```

**Пример 2**. Увеличим только четные элементы списка целых чисел на единицу:

```java
var list = new ArrayList<>(List.of(1, 2, 3, 4));
list.replaceAll(i -> i % 2 == 0 ? ++i : i);
list.forEach(System.out::print); //1335
```

  

#### void sort(Comparator<E> c)

Уже известный нам метод сортировки списка. Но т.к. тоже принимает параметром лямбда-выражение – упомянут здесь.

### Map

#### void forEach(BiConsumer<K, V> action)

Метод, по назначению совпадающий с _forEach()_ в _Iterable_, но с отличающейся сигнатурой. В случае с _Map_, данный метод работает с лямбдой, имеющей два параметра – ассоциированные с ключом и значением соответственно.

**Пример**. Выведем конкатенацию ключей и значений в мапе, разделяя разные пары пробелом:

```java
var map = new HashMap<Integer, String>();
map.put(1, "a");
map.put(2, "b");
map.put(3, "c");
map.put(4, "d");

map.forEach((k, v) -> System.out.print(k + v + " ")); //1a 2b 3c 4d
```

  

#### void replaceAll(BiFunction< K, V, V> function)

Метод схож с _replaceAll()_ для списка. Принимает параметром лямбда-выражение (входные параметры – ключ и значение), результат выражения – измененное значение для ключа.

Доработаем **пример** выше. Если значение ключа – четное, значение должно замениться на конкатенацию двух изначальных значений:

```java
var map = new HashMap<Integer, String>();
map.put(1, "a");
map.put(2, "b");
map.put(3, "c");
map.put(4, "d");

map.replaceAll((k, v) -> k % 2 == 0 ? v + v : v);
map.forEach((k, v) -> System.out.print(k + v + " ")); // 1a 2bb 3c 4dd
```

#### V compute(K key, BiFunction<K, V, V> remappingFunction)

Метод _compute()_ позволяет изменить значение по заданному ключу. Отдаленно похож на _replaceAll()_. Но если последний меняет (позволяет изменить) все значения в мапе, то _compute()_ – только для заданного ключа.

Возвращает измененное значение.

Важная особенность метода заключается в том, что если ключа, переданного первым параметром, не существует, лямбда-выражение все равно попытается отработать. При этом параметр, отвечающий за _value_ будет иметь значение _null_. Таким образом, в результате может как появиться новая пара ключ-значение в мапе, так и упасть _NPE_:

```java
var map = new HashMap<Integer, String>();
map.compute(1, (k, v) -> v + v); // мапа теперь содержит пару key=1,
                                 // value="nullnull"
map.compute(2, (k, v) -> v.getClass().toString()); // NPE – нельзя 
                                           // вызвать getClass() у null

```

  

#### V computeIfAbsent(K key, Function<K, V> mappingFunction)

Похож на _compute()_, но лямбда-выражение будет вызвано только если в данный момент отсутствует значение по текущему ключу.

Возвращает итоговое значение по ключу. Т.е. вернет созданное значение, если ранее оно отсутствовало или старое, если существовало.

На первый взгляд, это достаточно странная логика. Но она имеет смысл и _computeIfAbsent()_, наверно, самый популярный из _compute_\-методов _Map_.

**Пример**. Допустим, мы имеем список чисел и нам требуется представить его как _Map<Boolean, List<Integer>>_, разделив по четности. Мы можем без особых проблем решить данную задачу через Stream API. Но как будет выглядеть ее решение с использованием только методов коллекций?

```java
Map<Boolean, List<Integer>> intsByParity = new HashMap<>();

List.of(1, 2, 3, 4)
  .forEach(i -> {
    if (i % 2 == 0) {
      //Если список четных не существует - создаем его
      intsByParity.computeIfAbsent(true, k -> new ArrayList<>())
        .add(i); // добавляем элемент
    } else {
      //Если список нечетных не существует - создаем его
      intsByParity.computeIfAbsent(false, k -> new ArrayList<>())
        .add(i); // добавляем элемент
    }
});

System.out.println(intsByParity); //{false=[1, 3], true=[2, 4]}
```

Впрочем, для задачи, описанной выше, рекомендую использовать Stream API. Хотя сама концепция задачи встречается достаточно часто и периодически решается примерно так, как показано выше. Иногда это связано с неумением использовать Collector’ы, иногда – с какими-то более объективными причинами, по которым решено написать решение в практически императивном стиле. В последнем случае, зачастую, _forEach()_ будет заменен на полноценный цикл.

#### V computeIfPresent(K key, BiFunction<K, V, V> remappingFunction)

Думаю, вы уже догадались, в чем суть. _compute()_, который отработает только для существующих пар ключ-значение. Более того, если результатом лямбда-выражения из параметра окажется _null_ – пара с данным ключом будет удалена.

Пару очевидных примеров:

```java
Map<Integer, String> map = new HashMap<>();
map.put(1, "a");
map.put(2, "b");

map.computeIfPresent(1, (k, v) -> v + v); // key=1, value="aa"

map.computeIfPresent(2, (k, v) -> null);  // пара с ключом 2 удалена

map.computeIfPresent(3, (k, v) -> v + v); // по ключу 3 не существует 
                                          // значения, лямбда не будет 
                                          // вызвана

map.forEach((k, v) -> System.out.print(k + v + " ")); // 1aa
```

  

#### V merge(K key, V value, BiFunction<V, V, V> remappingFunction)

Данный метод был упомянут [при разборе Collector’а _toMap()_](/Stream-API-collect-Collector-Collectors-03-17#toMap()).

Суть метода сводится к тому, чтобы объединить два значения по одному ключу.

Актуальность метода в том, что повторный _put()_ по существующему ключу просто заменит старое значение на новое. Но иногда логика предполагает объединение этих значений. Тогда и приходит на помощь метод _merge()_. Лямбда-выражение, передаваемое параметром, ждет на вход старое и новое значения для данного ключа, а результатом станет объединенное значение.

Сам метод _merge()_ тоже вернет новое значение.

**Пример**. У нас есть некая мапа сумм. Т.е. по каждому ключу существует некоторая сумма, которая пополняется в каких-то обстоятельствах.

В таком случае, каждое добавление суммы в императивном стиле выглядит примерно так:

```java
sumMap.put(someKey, sumMap.get(someKey) + newSum);
```

Выглядит своеобразно, не говоря о возможном _NPE_, если мапа не содержит значения по ключу.

Можно вспомнить про _compute()_ и написать что-то вроде этого:

```java
sumMap.compute(someKey, (k, v) -> v == null ? newSum : v + newSum);
```

Но, в конечном итоге, можно использовать _merge()_:

```java
sumMap.merge(someKey, newSum, (oSum, nSum) -> oSum + nSum);
```

Результат будет аналогичен предыдущему, но лаконичнее и без ручных проверок на _null_.

Безусловно, пример очень утрированный и в случае с банальной суммой не так важно, как именно мы ее добавим. Но в качестве _value_ могут быть коллекции или иные составные типы, тогда первые два подхода могут оказаться крайне неудобными в использовании.

На этом мы завершаем разбор использования функциональных интерфейсов в коллекциях.

#### Итоги раздела ФП или «есть ли жизнь после Stream API?»

В данном пункте нас ждет две новости: хорошая и многообещающая.

Хорошая заключается в том, что мы изучили все основные концепции, на которых построено ФП в Java вне зависимости от особенностей реализации. По крайней мере, широкоизвестная его часть. К таким концепциям можно отнести **лямбда-выражения** и **функции высшего порядка** (по сути, это две части одного целого), а также **монады**.

Лямбда-выражения и функции высшего порядка будут преследовать вас повсюду. Это элемент ФП, который в Java давно живет даже в глубоко императивных проектах, без претензий на функциональный стиль. Разобранные выше методы коллекций тому подтверждение. Скорее всего, рано или поздно вы столкнетесь с желанием (или необходимостью) реализации задачи через собственные функциональные интерфейсы или, что более вероятно – с использованием собственных функций высшего порядка.

Но есть куда более интересная и разнообразная в плане реализаций концепция: монады.

Пока мы знакомы с двумя из них – _Stream_ и _Optional_. В следующем разделе – многопоточности – познакомимся с третьей – _CompletableFuture_.

  

Но это все находится в рамках JDK. Выходя за его пределы, рекомендую ознакомиться (для новичков – узнать, что существует) с реактивным программированием в Java. [_RxJava_](https://github.com/ReactiveX/RxJava), [_Project Reactor_](https://projectreactor.io/), [_Spring WebFlux_](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html) – мир реактивного программирования достаточно интересен и позволяет взглянуть на многие привычные вещи под совершенно иным углом. В контексте реактивного программирования придется существенно пересмотреть стек классического Java-приложения – от замены драйвера работы с БД до замены сервера, на базе которого работает приложение. Не говоря о смежных технологиях, которые можно использовать при реактивном подходе.

К сожалению, у меня нет заготовленных материалов по этому разделу, поэтому все что могу предложить – материал, с которого началось мое знакомство с концепцией реактивного программирования: [https://medium.com/@kirill.sereda/reactive-programming-reactor-%D0%B8-spring-webflux-3f779953ed45](https://medium.com/@kirill.sereda/reactive-programming-reactor-%D0%B8-spring-webflux-3f779953ed45)

  

Кроме того, ФП встречается в более узконаправленных инструментах, иногда – совсем неожиданно для разработчика. Так, классическим для меня (возможно, потому что было первым и сильно пошатнувшим уверенность в широте собственных знаний) примером является _Kafka Streams._

Нет, к Францу Кафке это отношения не имеет, но напрямую относится к [_Apache Kafka_](https://kafka.apache.org/) – брокеру сообщений, который, полагаю, известен большинству бэкенд-разработчиков вне зависимости от языка программирования.

Опять же, крайне рекомендую для ознакомления, если _Apache Kafka_ не режет слух новизной. Тут, к сожалению, без русскоязычной ссылки – у меня сложилось впечатление, что топ выдачи гугла со статьями для новичков – кривая калька с Baeldung, поэтому лучше сразу его: [https://www.baeldung.com/spring-boot-kafka-streams](https://www.baeldung.com/spring-boot-kafka-streams)

Для тех, кто после «_CompletableFuture_» перестал понимать, что происходит: ФП в Java давно заняло достаточно обширный ряд ниш и в данных направлениях стоит развиваться, если вам удалось «прочувствовать» Stream API. Как минимум, я рекомендую вернуться к абзацам выше, когда почувствуете себя уверенней. Кроме функциональных возможностей, ФП хорош тем, что позволяет взглянуть на разработку «по-новому», попытаться думать иначе. А закостенелость во взглядах и подходах – один из главных врагов программиста.

С теорией на сегодня все!

![](../../commonmedia/footer.png)

Переходим к практике. Не используйте Stream API для решения описанных задач:

#### Задача 1

Используя классы из практики к [уроку 57](/Stream-API-collect-Collector-Collectors-Praktika-03-17), реализуйте метод, принимающий на вход список сотрудников и возвращающий самого старшего обладателя каждого имени.

  

#### Задача 2

Используя классы из практики к [уроку 57](/Stream-API-collect-Collector-Collectors-Praktika-03-17), реализуйте метод, принимающий на вход список сотрудников и возвращающий список обладателей каждого имени.

  

#### Задача 3

Используя классы из практики к [уроку 57](/Stream-API-collect-Collector-Collectors-Praktika-03-17), реализуйте метод, принимающий на вход список сотрудников и возвращающий суммарный возраст обладателей каждого имени. Не используйте _Map.merge()_.

  

Если что-то непонятно или не получается – welcome в комменты к посту или в лс:)

Канал: [https://t.me/+relA0-qlUYAxZjI6](https://t.me/+relA0-qlUYAxZjI6)

Мой тг: [https://t.me/ironicMotherfucker](https://t.me/ironicMotherfucker)

_Дорогу осилит идущий!_