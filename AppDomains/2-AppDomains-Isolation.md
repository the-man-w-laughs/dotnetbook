## Изоляция

Как я уже говорил, нам нет никакой необходимости делать изоляцию по памяти: т.к. контроль над исполнением кода (путем его самостоятельной компиляции) дает домену все возможности для контроля всего приложения. Как это достигается? Домен - это некий изолирующий контейнер. Потому, если мы имеем дело с загружемыми библиотеками, было бы логично если бы эти библиотеки загружались бы не в общее для всех место, а в эту песочницу. Теперь смотрите: если в таком раскладе мы имеем один и тот же тип:

```csharp
class SomeType
{
    public int X { get; set; }
}
```

Который мы попробуем загрузить в два домена, то мы получим дважды скомпилированный код. Что это будет означать? Что у экземпляров этих типов адрес таблицы виртуальных методов совпадать не будет:

```csharp
// В первом домене:
var x = new SomeType();
var handle1 = x.GetType().TypeHandle;

// Во втором домене:
var x = new SomeType();
var handle2 = x.GetType().TypeHandle;

// где-то в общем коде
Console.WriteLine(handle1 == handle2);
// --> false
```

Это значит что если мы получим каким-либо образом из второго домена ссылку на объект и в первом домене попробуем привести к типу `SomeHandle`, то ничего не получится: типы не совпадут. Получается, что загрузив код в разные домены вместо загрузки в некое междоменное пространство `CLR` строит первый барьер защиты. Второй барьер защиты: нам необходимо организовать взаимодействие между доменами: быть полностью изолированными - так себе возможности и хочется иметь не полную изоляцию, а контролируемое взаимодействие. Интереснее получать отдавать некие данные в плагин для неких расчетов или действий и получать результаты работы - назад. Это можно организовать при помощи прокси-методов. Посмотрим на пример такого вызова с использованием псевдокода:

```csharp
// Мы получаем ссылку на объект из второго домена в первом
var objectFromSecondDomain = GetObjectFromDomain(domain2);

// И пробуем вызвать метод (как мы это обычно делаем)
var data = objectFromSecondDomain.Method();

// Что происходит на самом деле:
// Вместо метода вызывается невидимый для нас прокси метод,
// объявленный где-то в CLR, а не в самом типе:
var data = objectFromSecondDomain.MethodToDomainProxy();
private SomeType MethodToDomainProxy()
{
  if(CurrentDomain == Domain2){
    return objectFromSecondDomain.Method();
  }

  // Который приводит ссылку к необходимому типу из второго домена
  var castedType = (SomeType_FromDomain2)(objectFromSecondDomain);
  // сохраняем ссылку на наш домен и выставляем Domain 2 как активный
  var priorDomain = CurrentDomain;
  CurrentDomain = Domain2;
  // вызывает метод
  var result = castedType.Method();
  // восстанавливаем ссылку на текущий домен
  CurrentDomain = priorDomain;
  // и проверяет объект, который возвращен из метода на безопасность
  if(!CLR.VerifyResultForSafety(result))
  {
      throw CLR.GetExceptionForUnsafeCall();
  }
  // если все хорошо, возвращает результат
  return result;
}
```

Хорошо: теперь мы знаем обо всех вызовах, которые происходят в системе. Однако, это накладывает огромные расходы: нам ведь каждый раз необходимо понимать, что мы вызываем метод именно в другом домене, а не в текущем. А таких вызовов будет большинство: междоменного взаимодействия очень мало. Значит, чтобы такие расходы ложились бы только на необходимо ввести некий специальный тип, при наследовании от которого CLR поймет что он будет участвовать в междоменном взаимодействии и создаст прокси-методы только для него. Тип этот - `MarshalByRefObject` и это решение также создаст еще один барьер безопастности: вы не можете вызывать методы, скомпилированные для других доменов напрямую: только для `MarshalByRefObject` типов.

Что мы имеем на данный момент? Во-первых если наш код загружен в два и более доменов, то скомпилирован он будет соответствующее число раз, а значит с точки зрения типов один и тот же тип, скомпилированных для нескольких доменов - это несколько разных типов. Далее, вводятся проверки для вызовов методов из одного домена в другой. Можно назвать это словом "таможня". При пересечении границы все что проходит через такую вот таможню должно досматриваться: этим действием контролируется что все ограничения соблюдены. Ну и чтобы CLR понимала, какие типы могут участвовать во взаимодействии между доменами проходить границу могут не все типы, а только те, которые унаследовали `MarshalByRefObject`. Достаточно ли этого? Наверное, нет: мы отдаем в метод, скомпилированный для другого домена некую ссылку на свой объект, тот его принимает, но в другом домене работать с таким доменом нет никакой возможности: его система типов отлична и приведение типов работать не будет. Как тогда организовать взаимодействие?

Решения для такой проблематики два. Перовое решение - необходимо сделать так, чтобы передаваемые данные полностью бы отражали содержание объекта, но при этом не были бы самим объектом - раз. И при этом были бы достаточно простыми чтобы не представлять опасности - два. Такими данными язвяются сериализованные данные:

  - При сериализации мы все превращаем в условную строку. Т.е. не можем передать ссылку на объект, обладающий повышенными привелегиями;
  - При десериализации мы можем сформировать объекты только из того что было передано через условную строку: сериализованный объект. Ничего опасного мы оттуда получить не сможем.

Второе решение - организовать средство повышенного доверия к передаваемым между доменами данным. Такого доверия, что можно не волнуясь отдавать без всякой сериализации и десериализации ссылки на объекты в соседний домен и не волноваться что это как-то повлияет на безопасность. Чтобы решить эту задачу, был разработан FullTrust, который может быть применен к отдельным сборкам домена.

Касательно Shared AppDomains мы поговорим немного позже: слово за слово мне придется написать о многих вопросах. А потому эта тема поставлена на временную паузу. Сейчас же попробуем обосновать, создали ли мы идеальную песочницу:

  - Итак, для начала мы рассмотрели понятие домена как места, создающего изоляцию в вопросах исполнения кода. Внешние библиотеки загружаются отдельно в каждый их доменов и нет возможности кода одного домена работать с кодом другого, т.к. указатели на таблицы виртуальных методов будут разными;
  - Из-за этого возникает изоляция по памяти: объекты типов `домена 2` не могут попасть в методы `домена 1` по ранее указанной причине;
  - Чтобы этого не происходило между точкой вызова метода и его реальным вызовом встает проски-функция, которая проверяет параметры метда и возвращаемое значение на корректность.