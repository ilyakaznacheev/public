#Как сделать экран редактирования таблицы для Fiori за 20 минут

С появлением новой модели разработки приложений изменился и подход к работе с данными в системе. Однако в свете перемен разработчику требуется механизм, который позволит быстро реализовать такую простую и частую задачу, как возможность редактирования пользователем записей настроечной таблицы.

Начиная с SAP NW версии 7.5 эту задачу можно решить с применением всего двух инструментов - ABAP CDS View и Smart Templates.

В этой заметке я расскажу как быстро и просто создать такое приложение, используя фактически только CDS, не затрагивая ни ABAP, ни UI5, за пять простых шагов.

##Шаг 1. Создание таблицы

Создаем обычную кастомайзинговую таблицу (или берем существующую). Таблица будет редактироваться пользователем в системе, поэтому выбирается позволяющий это класс. Если вы хотите создать таблицу, изменения в которой будут переноситься транспортными запросами, то и приложение на Fiori для такой таблицы не нужно, данные будут вносить консультанты через sm30.

![](https://habrastorage.org/webt/ft/-p/40/ft-p40-odomkkqjvhp2u0mibmao.png)

Создать таблицу можно как в графическом, так и в текстовом редакторе, я предпочитаю для этой цели графический, так как он предохраняет от ошибок и достаточно удобен.

##Шаг 2. Написание Business Object CDS View

В соответствии с [новым подходом к разработке для Fiori](https://help.sap.com/viewer/cc0c305d2fab47bd808adcad3ca7ee9d/7.51.4/en-US/3b77569ca8ee4226bdab4fcebd6f6ea6.html) для транзакционного приложения необходимо создать бизнес-объект. В 7.5 это можно сделать при помощи обычного CDS View с набором определенных аннотаций.

Для описания бизнес-объекта используются аннотации ObjectModel.

- ```modelCategory: #BUSINESS_OBJECT``` - указывает, что CDS является описанием бизнес-объекта;
- ```compositionRoot: true``` - указывает, что view является корневым узлом бизнес-объекта (в нашем случае и единственным);
- ```transactionalProcessingEnabled: true``` - включает поддержку транзакционной обработки данных (возможность вносить изменения);
- ```semanticKey``` - список ключевых полей;
- ```writeActivePersistence``` - таблица, в которой хранятся данные узла;
- ```createEnabled```, ```deleteEnabled```, ```updateEnabled``` - разрешенные операции над данными объекта;
```mandatory``` - поля, обязательные для заполнения, будут помечены соответствующим образом в созданном приложении.

Полный список аннотаций можно увидеть в [описании модели](https://help.sap.com/viewer/cc0c305d2fab47bd808adcad3ca7ee9d/7.51.4/en-US/896496ecfe4f4f8b857c6d93d4489841.html).

В итоге получаем такую CDS:

```sql
@AbapCatalog.sqlViewName: 'ZTEST_I_STRKEP'
@AbapCatalog.compiler.compareFilter: true
@AccessControl.authorizationCheck: #CHECK
@EndUserText.label: 'Кладовщики'

@ObjectModel: {
  modelCategory: #BUSINESS_OBJECT,
  transactionalProcessingEnabled: true,
  compositionRoot: true,
  semanticKey: ['Plant', 'Storage'],
  writeActivePersistence: 'ZTEST_STOREKEEPR',
  createEnabled: true,
  deleteEnabled: true,
  updateEnabled: true
}

define view Ztest_I_Storekeeper
  as select from ztest_storekeepr
{
      mandt,
  key plant     as Plant,
  key storage   as Storage,
      @ObjectModel.mandatory: true
      firstname as FirstName,
      @ObjectModel.mandatory: true
      lastname  as LastName,
      phone     as Phone
}
```
При активации будет создан бизнес-объект с таким же названием.
![](https://habrastorage.org/webt/-l/j8/m1/-lj8m1mrj8yanuhn_ofkxevupps.png)

*В CDS будет показано, что сгенерирован бизнес-объект*

![](https://habrastorage.org/webt/ig/5c/3x/ig5c3xsogislkvr5b8crc6xkaay.png)

*Созданный бизнес-объект*

![](https://habrastorage.org/webt/hw/sz/4x/hwsz4xj28csju1jxqcva5dbtxj0.png)

*Просмотр корневого узла бизнес-объекта*

Обратите внимание на то, что функционал и возможности бизнес-объектов, сгенерированных на основе CDS View, несколько урезан относительно обычных бизнес-объектов, что в некоторой степени ограничивает их использование для реализации особенно сложной логики. Однако для простых задач, подобных этой, их функционала более чем хватает.

Этот бизнес-объект будет обслуживать изменения в табличных данных. При необходимости ему можно добавить проверок или различных действий (которые также можно вынести в Fiori приложение при помощи Smart Templates), но в рамках текущей задачи это не нужно.

##Шаг 3. Написание Consumption CDS View

Следующий шаг - написание т.н. Consumption View, который описывает структуру данных, передаваемую в OData-сервис и затем на фронтенд. В этом view, помимо самих полей, описываются также параметры их отображения в приложении Fiori, и все оформление и функционал пользовательского интерфейса.

Этот шаг наиболее сложный и многообразный, так как именно здесь описываются все характеристики создаваемого приложения. Весь внешний вид и поведение описываются посредством аннотаций. Описание примеров использования аннотаций для реализации тех или иных задач изложен в [справочнике](https://help.sap.com/viewer/cc0c305d2fab47bd808adcad3ca7ee9d/7.5.9/en-US/d57438669aae4d0083ce767b9505ca48.html). Там же можно найти и список всех аннотаций с описанием. Здесь же мы рассмотрим минимальный набор для построения простого, но удобного интерфейса.

###UI - аннотации разметки интерфейса

- ```typeName```, ```typeNamePlural``` - заголовок списка в единственном и множественном числе в зависимости от количества записей;
- ```title.value``` - заголовок на экране просмотра позиции, получается из указанного поля cds;
- ```lineItem.position``` - позиция поля в строке таблицы;
- ```lineItem.label``` - название поля в строке таблицы (если не указывать, отобразится текст из домена);
- ```identification.position``` - позиция на экране просмотра позиции;
- ```identification.label``` - название поля на экране просмотра позиции (если не указывать, отобразится текст из домена);
- ```selectionField.position``` - позиция поля в строке поиска, 
- ```hidden: true``` - скрытое поле, оно не будет отображаться в таблице и детальном экране. 

В этом примере мы добавили поле FullName, состоящее из имени и фамилии кладовщика, это поле используется только для отображения в заголовке второго экрана. Поэтому его необходимо пометить как ```@UI.hidden: true```, чтобы оно не выводилось наряду с другими полями. Также оно не может быть изменено, так как не отражает реально существующих в БД полей, а является производным от них, поэтому оно помечено аннотацией ```@ObjectModel.readOnly: true```, указывающей на то, что поле доступно только для чтения.

###Search - аннотации нечеткого текстового поиска

- ```searchable: true``` - включает механизм поиска для CDS;
- ```defaultSearchElement: true``` - включает поле в поисковый алгоритм;
- ```fuzzinessThreshold``` - показатель точности поиска по полю;
- ```ranking``` - градация “важности” поля при поиске.

При включении этого механизма в заголовке табличной части появляется дополнительное поле поиска, в котором можно искать по данным указанных полей. Это довольно удобно для текстовых полей, когда пользователь вводит часть текста чтобы найти подходящие записи.

###ObjectModel - управление изменениями

- ```transactionalProcessingDelegated: true``` - обработка изменений делегирована нижележащей CDS, в нашем случае это CDS с описанием бизнес-объекта;
- ```compositionRoot: true``` - указывает, что view является отображением корневого узла бизнес-объекта;
- ```semanticKey``` - список ключевых полей;
- ```createEnabled```, ```deleteEnabled```, ```updateEnabled``` - разрешенные операции над данными объекта;
- ```readOnly: true``` - указывает, что поле доступно только на чтение.

###Средства поиска

Для того, чтобы при вводе значения было доступно средство поиска, необходимо сделать следующее:

1. добавить ассоциацию с кардинальностью 1 к 1 на CDS со списком полей;
2. объявить ассоциацию в списке CDS;
3. добавить искомому полю аннотацию  ```@Consumption.valueHelp: '...'```, где в кавычках указывается название ассоциации.

После этого средство поиска становится доступно в приложении.

![](https://habrastorage.org/webt/bh/xs/ax/bhxsaxbd-hbo_hmyafbrgpec29w.png)

*Средства поиска доступны для полей Plant и Storage*

###Тексты

Для того, чтобы добавить к полю текстовое описание его значений, необходимо сделать следующее:

1. добавить ассоциацию с кардинальностью 1 к 1 на CDS с текстовыми значениями;
в CDS должно быть минимум одно поле с аннотацией ```@Semantics.text: true``` - в качестве текста будет выбрано первое поле с такой аннотацией;
2. объявить ассоциацию в списке CDS;
3. добавить искомому полю аннотацию ```@ObjectModel.text.association: '...'```, где в кавычках указывается название ассоциации.

После этого тексты появятся рядом с данными соответствующих полей.

![](https://habrastorage.org/webt/61/4s/ac/614sacflj3lf0ftv6-knhxhyvem.png)

*Тексты на экране просмотра таблицы*

![](https://habrastorage.org/webt/qf/q7/6n/qfq76n1wt2djranj5kr0mbkhk7c.png)

*Тексты на экране просмотра записи*

В итоге получаем такую CDS:

```sql
@AbapCatalog.sqlViewName: 'ZTEST_C_STRKEP'
@AbapCatalog.compiler.compareFilter: true
@AccessControl.authorizationCheck: #CHECK
@EndUserText.label: 'Кладовщики'

@Search.searchable: true

@ObjectModel: {
  transactionalProcessingDelegated: true,
  compositionRoot: true,
  semanticKey: ['Plant', 'Storage'],
  createEnabled: true,
  deleteEnabled: true,
  updateEnabled: true
}

@UI.headerInfo: {
  typeName: 'Кладовщик',
  typeNamePlural: 'Кладовщики',
  title.value: 'FullName'
}

define view Ztest_C_Storekeeper
  as select from Ztest_I_Storekeeper as Storekeeper

  association [1..1] to I_Plant           as _Plant           on  _Plant.Plant = Storekeeper.Plant
  association [1..1] to I_StorageLocation as _StorageLocation on  _StorageLocation.Plant           = Storekeeper.Plant
                                                              and _StorageLocation.StorageLocation = Storekeeper.Storage
{
      @UI: {
        identification.position: 10,
        lineItem.position: 10,
        selectionField.position: 10
      }
      @Consumption.valueHelp: '_Plant'
      @ObjectModel.text.association: '_Plant'
  key Plant,

      @UI: {
        identification.position: 15,
        lineItem.position: 20,
        selectionField.position: 20
      }
      @Consumption.valueHelp: '_StorageLocation'
      @ObjectModel.text.association: '_StorageLocation'
  key Storage,

      @UI: {
        identification.position: 30,
        identification.label: 'Имя',
        lineItem.position: 30,
        lineItem.label: 'Имя'
      }
      @Search: {
        defaultSearchElement: true,
        fuzzinessThreshold: 0.8,
        ranking: #HIGH
      }
      @Semantics.name.givenName: true
      FirstName,

      @UI: {
        identification.position: 40,
        identification.label: 'Фамилия',
        lineItem.position: 40,
        lineItem.label: 'Фамилия'
      }
      @Search: {
        defaultSearchElement: true,
        fuzzinessThreshold: 0.8,
        ranking: #HIGH
      }
      @Semantics.name.familyName: true
      LastName,

      @UI: {
        identification.position: 50,
        identification.label: 'Номер телефона',
        lineItem.position: 50,
        lineItem.label: 'Телефон'
      }
      @Semantics.telephone.type: #WORK
      Phone,

      @UI.hidden: true
      @ObjectModel.readOnly: true
      concat_with_space(FirstName, LastName, 1) as FullName,

      _Plant,
      _StorageLocation
}
```

В данном примере CDS для средства поиска и CDS для текстовых значений совпадают, но это может быть и не так - просто указывайте соответствующие CDS где надо и не забудьте добавить их в конец, чтобы приложение могло получить к ним доступ через сервис.

Все тексты, написанные в ассоциациях, сохраняются как тексты на языке оригинала CDS, и доступны для перевода через транзакцию se63.

##Шаг 4. Генерация и активация OData-сервиса

Для генерации сервиса необходимо добавить следующую аннотацию: 

```sql
@OData.publish: true
```

И активировать CDS.

После этого необходимо открыть транзакцию /IWFND/MAINT_SERVICE и добавить созданный сервис (“Добавить сервис” -> “Получить сервисы” -> нажать на нужный сервис -> ввести пакет и необходимые изменения -> “ОК”). 

![](https://habrastorage.org/webt/mh/qm/bq/mhqmbqpoh8uoibcxzkmuc9yiosm.png)

*Транзакция /IWFND/MAINT_SERVICE - добавление сервиса*

![](https://habrastorage.org/webt/9g/n4/5z/9gn45zhvlebuyjzensr4qpwfqb8.png)

*Ввод параметров добавляемого сервиса*

После этого сервис будет доступен для использования на фронтенде.

 ![](https://habrastorage.org/webt/ls/p4/xa/lsp4xam8w91hntp81ngfp7hk6lo.png)

*В CDS показано, что OData сервис сгенерирован и активен*

##Шаг 5. Создание Fiori приложения на основе Smart Template

На этом шаге остается малое - создать приложение на основе Smart Template. Изменять или дорабатывать фронтенд не придется - Smart Template загрузит всю необходимую информацию из аннотаций в CDS.

Нужно зайти в WebIDE и создать новый проект:

1. “New”;
1. "Project from template”;
1. “List Report Application”;
1. указать название и описание приложения;
1. выбрать источник данных;
1. указать систему;
1. выбрать сервис из списка доступных в этой системе
1. выбрать источник метаданных (достаточно того, что носит название как у CDS);
1. указать binding - выбрать соединение OData (будет называться так же, как CDS);
1. “ОК”.

![](https://habrastorage.org/webt/wn/h4/-c/wnh4-ctbqkyrz1szwplx6wak8ey.png)

*Выбор сервиса при создании приложения*

После этих действий весь необходимый приложению код будет сгенерирован в директории проекта, и его можно будет запустить.

![](https://habrastorage.org/webt/75/_d/on/75_dony6v-ip-qdrb94iwxbsc24.png)

*Плитка в Fiori Launchpad*

![](https://habrastorage.org/webt/bt/q-/a0/btq-a0ma1t2uxmqsy7ojcrng4c4.png)

*Первый экран приложения*

![](https://habrastorage.org/webt/8b/k4/hm/8bk4hmmftu1pogqwekhkd3spyi0.png)

*Второй экран приложения - просмотр*

![](https://habrastorage.org/webt/a-/9i/wy/a-9iwyw0rusu99xtyzqyb9ers2y.png)

*Второй экран приложения - создание*

![](https://habrastorage.org/webt/k7/tq/u4/k7tqu4abzv3fsuzkrscbi_d5ysm.png)

*Средство поиска поля Storage*

После этого приложение необходимо задеплоить обычным образом, и оно будет доступно в качестве плитки в ланчпаде пользователям системы.

*P.S.: Если хотите добавить кнопку выгрузки в эксель, [как это сделать описано здесь](https://blogs.sap.com/2018/04/23/new-excel-export-functionality-available).*