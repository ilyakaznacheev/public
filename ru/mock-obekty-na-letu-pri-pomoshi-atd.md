#Mock-объекты на лету при помощи ATD

В юнит тестировании распространена практика [dependency injection](https://ru.wikipedia.org/wiki/Внедрение_зависимости) (DI), которая позволяет вынести внешние зависимости в отдельные сущности, которые передаются в объект в тот или иной момент через интерфейс. В тесте эти сущности заменяются заглушками с заранее известным поведением (mock-объект). Таким образом тестируемый объект изолируется от внешних зависимостей.

В NW 7.4 SP 9+ появился инструмент, позволяющий создавать mock-объекты, имплементирующее определенный интерфейс, прямо в рантайме. Это достаточно удобно при тестировании, так как не нужно прописывать имплементацию интерфейсов в mock-объектах для DI вручную, это можно сделать через вызов нескольких методов.

Взглянем на этот инструмент более внимательно.

##ABAP Test Double Framework

В качестве основного класса фреймворка выступает `cl_abap_testdouble`. Этот класс позволяет конструировать логику работы mock-объекта путем последовательной настройки того, как отработает каждый используемый метод интерфейса. Это выглядит следующим образом.

###Создание mock-объекта для DI 

Вначале необходимо создать объект, имплементирующий нужный вам интерфейс. Это делается через вызов метода `create`, который возвращает ссылку на созданный объект. Объект необходимо явно привести к типу интерфейса:

```abap
DATA(lo_geo_double) = CAST zif_test_taxi_geo_service(
  cl_abap_testdouble=>create( 'zif_test_taxi_geo_service' )
).
```

В этом примере мы имеем некоторый интерфейс геосервиса для таксистов. Интерфейс имеет два метода - `get_path_length`, который возвращает длину пути между двумя точками, и `is_path_available`, который проверяет, можно ли проехать от одной точки до другой.

Для определения логики работы каждого из методов необходимо последовательно задать все параметры и логику работы метода, а потом вызвать метод со входными параметрами. Если логика отличается для разных входных параметров, и вы хотите это реализовать в одном mock-объекте, то настройку метода и последующий вызов необходимо повторить.

```abap
cl_abap_testdouble=>configure_call(
  lo_geo_double
)->returning( 1 ).

lo_geo_double->get_path_length(
  iv_from = VALUE #( x = 1 y = 1 )
  iv_to   = VALUE #( x = 11 y = 11 )
).

cl_abap_testdouble=>configure_call(
  lo_geo_double
)->returning( 2 ).


lo_geo_double->get_path_length(
  iv_from = VALUE #( x = 2 y = 2 )
  iv_to   = VALUE #( x = 12 y = 12 )
).
```

Здесь мы сконфигурировали логику работы метода `get_path_length` для двух наборов значений - для поездки из точки (1,1) в точку (11,11) рассчитаная длина составляет 1, а и точки (1,1) в точку (12,12) - 2. Если параметры для вызова метода не заданы, метод возвращает пустые значения.

Если не нужно обрабатывать входные параметры, то некоторые из них или сразу все можно игнорировать. При этом все равно необходимо вызвать сам метод после его настройки, при этом в него нужно передать входные, но так как метод их игнорирует, можно передать любые просто для соответствия синтаксису языка.

```abap
cl_abap_testdouble=>configure_call(
  lo_geo_double
)->ignore_all_parameters(
)->returning( abap_true ).

lo_geo_double->is_path_available(
  iv_from = VALUE #( x = 0 y = 0 )
  iv_to   = VALUE #( x = 0 y = 0 )
).
```

В этом примере метод `is_path_available` игнорирует любые входные параметры и всегда возвращает `abap_true`. Однако при вызове все равно нужно заполнить обязательные параметры метода. 

На этом все, *mock-объект готов к использованию*.
Допустим, у нас есть некоторый класс, использующий описанный геосервис сервис в методе:

```abap
METHOD get_ride_price.

  IF NOT go_geo_service->is_path_available(
    iv_from = iv_from
    iv_to   = iv_to
  ).
    RAISE EXCEPTION TYPE zcx_taxi_ride_not_possible.
  ENDIF.

  DATA(lv_ride_cost) = gv_meter_cost * go_geo_service->get_path_length(
    iv_from = iv_from
    iv_to   = iv_to
  ).

  IF lv_ride_cost < gv_minimun_price.
    rv_ride_cost = gv_minimun_price.
  ELSE.
    rv_ride_cost = lv_ride_cost.
  ENDIF.

ENDMETHOD.
```

Где `go_geo_service` содержит ссылку на интерфейс `zif_test_taxi_geo_service`, инстанция которого передается в конструктор класса, тем самым реализуя DI.

Чтобы написать юнит-тест для этого класса, не нужно даже иметь реализованную имплементацию интерфейса `zif_test_taxi_geo_service`, достаточно замокать его как показано выше. Таким образом, помимо прочего, это позволяет легче распределить объем разработок внутри команды - класс, использующий геосервис, может быть уже реализован, протестирован и задокументирован, когда разработка самого геосервиса еще даже не начиналась.

Вот как можно реализовать такой тест:

```abap
DATA(lo_cut) = NEW zcl_test_taxi_ride_calculator(
  iv_minimun_price = lv_min_price
  iv_meter_cost    = '1.5'
  io_geo_service   = lo_geo_double
).

DATA(lv_act1) = lo_cut->get_ride_price(
  iv_from = VALUE #( x = 1 y = 1 )
  iv_to   = VALUE #( x = 11 y = 11 )
).

DATA(lv_act2) = lo_cut->get_ride_price(
  iv_from = VALUE #( x = 2 y = 2 )
  iv_to   = VALUE #( x = 12 y = 12 )
).

cl_abap_unit_assert=>assert_equals(
  act = lv_act1
  exp = lv_min_price
  msg = |Wrong ride price { lv_act1 } but expected { lv_min_price }|
).

cl_abap_unit_assert=>assert_equals(
  act = lv_act2
  exp = lv_min_price
  msg = |Wrong ride price { lv_act1 } but expected { lv_min_price }|
).
```

Здесь в конструктор класса `zcl_test_taxi_ride_calculator` передается созданный и настроенный нами mock-объект интерфейса `zif_test_taxi_geo_service`, который дальше используется при выполнении тестовых вызовов метода класса.

Дальше мы проверяем, совпадает ли результат работы метода с ожидаемыми значениями, и, если нет, фреймворк юнит-тестов выдаст в лог тестирования сообщение об ошибке.

Полный пример можно посмотреть [здесь](https://gist.github.com/ilyakaznacheev/f71dcf483e169307401b5607f700d7e5).

Mock-объекты, создаваемые через `cl_abap_testdouble`, имеют довольно широкий функционал. Они позволяют настроить `changing` и `exporting` параметры, генерировать события и исключения, указывать, сколько раз метод может быть вызван ,и более сложные условия.

###Создание множества объектов для тестирования зависимой от них логики

Один из интересных шаблонов использования mock-объектов - необходимость протестировать работу логики, взаимодействующей с некой сущностью, то есть, когда на вход подается объект, и метод (или функция) что-то с его помощью делает. Для тестирования такого пришлось бы локально описывать множество имплементаций интерфейса, которые покрыли бы все варианты работы с ним, которые мы хотим протестировать. ATD позволяет сделать это на лету.

Допустим у нас есть некоторый интерфейс сущности, описывающей поездку на такси.

```abap
INTERFACE zif_test_taxi_ride
  PUBLIC .

  TYPES:
    BEGIN OF ENUM ts_rates,
      basic,
      business,
      vip,
    END OF ENUM   ts_rates.

  METHODS get_info
    EXPORTING
      ev_from           TYPE geo_point
      ev_to             TYPE geo_point
      ev_rate           TYPE ts_rates
      ev_passenger_name TYPE string.

  METHODS request_payment
    IMPORTING
      iv_ride_cost     TYPE paymnt
    RETURNING
      VALUE(rv_status) TYPE bool.

ENDINTERFACE.
```

Он используется в методе обработчика поездок для выполнения поездки и учета заработанных денег.

```abap
io_ride->get_info(
  IMPORTING
    ev_from           = DATA(lv_from)
    ev_to             = DATA(lv_to)
    ev_rate           = DATA(lv_rate)
    ev_passenger_name = DATA(lv_name)
).

DATA(lv_price) = go_ride_calculator->get_ride_price(
   iv_from  = lv_from
   iv_to    = lv_to
 ).

*  recalculate price with rate chosen
DATA(lv_rated_price) = lv_price * SWITCH int1( lv_rate
  WHEN zif_test_taxi_ride=>basic    THEN 1
  WHEN zif_test_taxi_ride=>business THEN 2
  WHEN zif_test_taxi_ride=>vip      THEN 3
).

*  request payment, if the passenger does not want to pay, raise an error
IF NOT io_ride->request_payment( iv_ride_cost = CONV #( lv_rated_price ) ).
  RAISE EXCEPTION TYPE zcx_taxi_ride_payment_denial
    EXPORTING
      textid         = zcx_taxi_ride_payment_denial=>zcx_taxi_ride_payment_denial
      passenger_name = lv_name.
ENDIF.

gv_revenue = gv_revenue + lv_rated_price.
```

Здесь метод получает информацию о поездке: точке ее начала и конца, тарифе и имени пользователя. Эта информация дальше используется для получения данных о поездке от геосервиса, расчета конечной стоимости, оплате, обработке случая, когда клиент отказывается оплатить, и учет полученных средств.

Для тестирования нам необходимо передавать в метод различные объекты “поездок”.

```abap
*  create ride mock 1
DATA(lo_ride_1) = CAST zif_test_taxi_ride(
  cl_abap_testdouble=>create( 'zif_test_taxi_ride' )
).

cl_abap_testdouble=>configure_call(
  lo_ride_1
)->set_parameter(
  name  = 'ev_from'
  value = VALUE  geo_point( x = 1 y = 1 )
)->set_parameter(
  name  = 'ev_to'
  value = VALUE  geo_point( x = 2 y = 2 )
)->set_parameter(
  name  = 'ev_rate'
  value = zif_test_taxi_ride=>basic
)->set_parameter(
  name  = 'ev_passenger_name'
  value = 'test name'
).
lo_ride_1->get_info( ).

cl_abap_testdouble=>configure_call(
  lo_ride_1
)->ignore_all_parameters(
)->returning( abap_true ).
lo_ride_1->request_payment( 1 ).


* ...
*  create as many objects as you want
* ...

*  create cut instance and execute test
DATA(lo_cut_1) = NEW zcl_test_taxi_ride_handler( go_ride_calculator ).
lo_cut_1->handle_ride( lo_ride_1 ).
DATA(lo_act_price_1) = lo_cut_1->get_revenue( ).

cl_abap_unit_assert=>assert_equals(
  act = lo_act_price_1
  exp = 100
  msg = |Wrong ride price { lo_act_price_1 } but expected { 100 }|
).


* ...
*  check each created object
* ...
```

В целях читабельности листинг сокращен.

В этом примере класс `zcl_test_taxi_ride_handler` также принимает `zif_test_taxi_geo_service` в конструктор (мы также настроили mock-объект для него заранее), а затем метод `handle_ride` выполняет некоторые действия с объектом интерфейса `zif_test_taxi_ride`. В этом примере чтобы протестировать различную работу класса нам необходимо создать набор тестовых mock-объектов и с их помощью выполнить проверку того, как тестируемая логика отрабатывает на каждом из них (или совокупности). Обратите внимание на то, как конфигурируются значения `exporting` параметров.

##Table driven tests

Еще один паттерн, который можно применить для тестирования - “Table driven tests”, когда входные данные теста и ожидаемые результаты описываются заранее в виде таблицы тестовых случаев, а затем по очереди выполняются. В данном случае при обходе таблицы тестовых значений для каждого случая создается mock-объект, конфигурируются вызовы его методов на основе данных из таблицы, создается тестируемый объект, тестируется с использованием созданного в данной итерации mock-объекта, затем результат сравнивается с ожидаемым значением.

```abap
* …
* declare test cases in table lt_cases
* …

LOOP AT lt_cases ASSIGNING FIELD-SYMBOL(<ls_case>).

*  build mock-object
  <ls_case>-object = CAST #(
    cl_abap_testdouble=>create( 'zif_test_taxi_ride' )
  ).

*  setup first method
  DATA(lo_conf) = cl_abap_testdouble=>configure_call( <ls_case>-object ).
  LOOP AT <ls_case>-params ASSIGNING FIELD-SYMBOL(<ls_param>).
    ASSIGN <ls_param>-value->* TO <lv_value>.
    CHECK sy-subrc = 0.
    lo_conf->set_parameter(
      name  = <ls_param>-name
      value = <lv_value>
    ).
  ENDLOOP.
  <ls_case>-object->get_info( ).

*  setup second method
  cl_abap_testdouble=>configure_call(
    <ls_case>-object
  )->ignore_all_parameters(
  )->returning( abap_true ).
  <ls_case>-object->request_payment( 0 ).

*  create cut instance and execute test
  DATA(lo_cut) = NEW zcl_test_taxi_ride_handler( go_ride_calculator ).
  lo_cut->handle_ride( <ls_case>-object ).

  DATA(lv_act_price) = lo_cut->get_revenue( ).

*  assert results
  cl_abap_unit_assert=>assert_equals(
    act = lv_act_price
    exp = <ls_case>-exp_price
    msg = |[{ <ls_case>-number }]: Wrong ride price { lv_act_price } but expected { <ls_case>-exp_price }|
  ).

ENDLOOP.
```

Полностью пример класса и тестов можно посмотреть [здесь](https://gist.github.com/ilyakaznacheev/2ef42ffc597257a1dfb95ef573fc41f0).

##Еще возможности

Помимо описанного функционала mock-объекты имеют и более продвинутые возможности. Например:

- настраивать ожидаемое количество вызовов метода mock-объекта при помощи метода `and_expect` (в случае неудачи генерирует исключение `CX_ATD_EXCEPTION`);
- задать более сложную логику работы метода mock-объекта при помощи метода `set_answer`, которому передается объект, имплементирующий интерфейс `if_abap_testdouble_answer` - вы можете реализовать свой кастомный обработчик, имплементирующий этот интерфейс, и использовать его вместо стандартного в качестве реализации логики этого метода mock-объекта;
- задать более сложную логику проверки входных параметров метода mock-объекта при помощи метода `set_matcher`, которому передается объект, имплементирующий интерфейс `if_abap_testdouble_matcher` - вы можете реализовать свою кастомную проверку входных значений, имплементирующую этот интерфейс, и использовать ее вместо стандартной в качестве реализации логики проверок для этого метода mock-объекта;

##Итого

В целом использование ATD для создания mock-объектов в рантайме хоть и выглядит несколько громоздким, тем не менее позволяет значительно упростить написание юнит-тестов для вашего кода и, что не менее важно, упростить поддержку и актуализацию этих тестов по мере разработки и появлениея связанных с этим изменений в тестируемой логике.