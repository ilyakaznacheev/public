# Контейнеры сообщений в ABAP

В ABAP реализован достаточно мощный механизм использования сообщений, в который интегрированы и локализация, и включение переменных в текст сообщений, передача этих сообщений в RFC, использование в классических и объектных исключениях, логирование, а также широкие возможности по выводу сообщений на различных вариантах GUI. При этом часто возникает необходимость передавать сообщения не поодиночке, а списком, например по завершении какого-то комплексного процесса, состоящего из шагов, либо при вызове программного модуля (RFC, BAPI и т.п.), возвращающего список сообщений.

Достаточно распространенной является практика использования таблицы со структурой BAPIRET в той или иной модификации. Как правило, таблица содержит не только сообщения, но и дополнительную информацию, как статусы, информация для логирования и другое.

Несмотря на простоту и распространенность этого решения, часто возникают похожие задачи по работе с данными из таких списков. Например получение наихудшего статуса, выбор сообщений с определенным статусом, добавление/удаление сообщений, логирование списка и прочее. Так или иначе на крупных проектах это приводит к необходимости создания некой обертки над списком сообщений, которая будет реализовывать весь функционал по работе с ним. Написание подобных классов - довольно увлекательный процесс, однако следует знать, что в стандарте уже существуют решения, способные покрыть почти все эти требования. О наиболее распространенных из них я расскажу в этой заметке.

## RECA

В стандартной поставке ABAP есть пакет RECA, включающий в себя множество утилитарных классов, решающих широкий спектр технических задач. Пакет входит в модуль RE-FX, который поставляется в составе SAP ERP, однако может быть найден и в других системах, например в EHS. Многие этих классов достойны отдельных статей, но сегодня мы поговорим об одном из них - `CL_RECA_MESSAGE_LIST`, а точнее его интерфейсе `IF_RECA_MESSAGE_LIST`, через который он и будет использоваться в программе.

Интерфейс имеет множество методов для работы со списком сообщений, для добавления сообщений из разных источников, для анализа сообщений и работы с логированием. Ниже я рассмотрю некоторые из них.

Инстанция создается через factory класс:

```abap
DATA:
  lo_message_list TYPE REF TO if_reca_message_list.

lo_message_list = cf_reca_message_list=>create( ).
```

В метод можно передать параметры BAL объекта, если вы хотите сохранять в него сообщения. Также класс `cf_reca_message_list` предоставляет инструментарий для поиска существующих BAL объектов.

`IF_RECA_MESSAGE_LIST` позволяет добавлять сообщения из различных источников: через указание номера сообщения, из объекта исключения, из таблицы BAPIRET, из sy, а также из другого объекта сообщений.

```abap
lo_message_list->add(
  id_msgty = 'S'
  id_msgid = 'RDA'
  id_msgno = 24
  id_msgv1 = 'test'
).

*CALL BAPI
lo_message_list->add_from_bapi(
  it_bapiret = lt_bapi_result
).

*CATCH cx_root INTO lo_exception.
lo_message_list->add_from_exception(
  io_exception = lo_exception
).

lo_message_list->add_from_instance(
  io_msglist = lo_message_list2
).

lo_message_list->add_symsg( ).
```

Стоит обратить внимание, что в интерфейсе нет метода, позволяющего добавлять сообщение из символьной переменной. Вероятно это связано с тем, что тексты в литералах являются плохим тоном в ABAP, так как не позволяют осуществлять перевод и затрудняют поддержку. Если вы один из тех, кто хардкодит тексты в строковые переменные, то при работе с этим объектом от такого подхода придется отказаться.

Этот контейнер удобно использовать как возвращаемый параметр метода, который выполняет цепочку действий, статус которых вы хотите в дальнейшем как-то использовать. Также удобно записывать сообщения, которые возвращают BAPI и другие ФМ, в частности те, что вызываются по RFC - зачастую они возвращают список сообщений именно в формате BAPIRET или близком к нему, который несложно сконвертировать в нужный.

```abap
DATA:
  lt_message_1 TYPE bapiret1_tab,
  lt_message_2 TYPE mmpur_message_list.

lo_message_list->add_from_bapi(
  it_bapiret = CORRESPONDING #( lt_bapi_result )
).

lo_message_list->add_from_bapi(
  it_bapiret = CORRESPONDING #( lt_message_2 MAPPING
    type        = msgty
    id          = msgid
    number      = msgno
    message_v1  = msgv1
    message_v2  = msgv2
    message_v3  = msgv3
    message_v4  = msgv4
  )
).
```

Класс также позволяет получить сообщения обратно в удобном виде - получить все сообщения в виде таблицы BAPIRET, получить первое или последнее сообщение, отфильтровать сообщения по типу. 

Помимо этого, класс позволяет получать различные аналитические данные по сообщениям, на основе которых удобно строить различные логигческие конструкции и проверки, не обращаясь раз за разом к самому списку напрямую. Например, можно получить статистику по всему списку сообщений, либо проверить, есть ли сообщения с определенным типом и есть ли сообщения вообще.

```abap
IF lo_message_list->has_messages_of_msgty( 'E' ).
*  error processing of types E and X
ENDIF.

DATA(ls_msg_statistics) = lo_message_list->get_statistics( ).

IF ls_msg_statistics-msg_cnt_e > 0 OR ls_msg_statistics-msg_cnt_a > 0.
*  error processing
ELSEIF ls_msg_statistics-msg_cnt_w > 0.
*  warning processing
ELSE.
*  success processing
ENDIF.
```

Ну и помимо прочего в контейнер встроен функционал по работе с BAL, например изменение заголовка, изменение уровня детализации сообщений и, конечно, сохранение. Пример использования:

```abap
lo_message_list = cf_reca_message_list=>create(
    id_object       = 'ZTEST'
    id_subobject    = 'SUBTEST'
    id_extnumber    = CONV #( sy-repid )
).

MESSAGE s000(00) INTO lv_dummy.
lo_message_list->add_symsg( ).
lo_message_list->store( ).
```

Для вывода сообщений класса на экран в классическом Dynpro можно воспользоваться ФМ `RECA_GUI_MSGLIST_POPUP`, который выводит сообщения на стандартном экране лога.

Контейнер сообщений с подобным функционалом находит множество вариантов применения. Помимо очевидных вариантов вроде возвращаемого параметра, контейнер можно использовать, например, как атрибут класса исключения, куда можно записать несколько сообщений, описывающих возникшую ошибочную ситуацию, либо включить подробное описание шагов процесса, предшествующего ошибке. При обработке исключения эти сообщения можно включить в контейнер другого класса исключения, если обработка ошибки будет продолжена на более высоком уровне по стеку вызова, или при окончательной обработке ошибки вывести эти сообщения на экран или добавить в лог.

```abap
CLASS zcx_random_error DEFINITION
  PUBLIC
  INHERITING FROM cx_static_check
  FINAL
  CREATE PUBLIC .

  PUBLIC SECTION.

    INTERFACES if_t100_dyn_msg .
    INTERFACES if_t100_message .

    METHODS constructor
      IMPORTING
        textid        LIKE if_t100_message=>t100key OPTIONAL
        previous      LIKE previous OPTIONAL
        msg_container TYPE REF TO if_reca_message_list OPTIONAL.
    METHODS get_msg_container
      RETURNING
        VALUE(ro_msg_container) TYPE REF TO if_reca_message_list .

  PRIVATE SECTION.

    DATA go_msg_container TYPE REF TO if_reca_message_list .
ENDCLASS.

CLASS zcx_random_error IMPLEMENTATION.
  METHOD constructor.
    CALL METHOD super->constructor
      EXPORTING
        previous = previous.
    CLEAR me->textid.
    IF textid IS INITIAL.
      if_t100_message~t100key = if_t100_message=>default_textid.
    ELSE.
      if_t100_message~t100key = textid.
    ENDIF.

    IF msg_container IS BOUND.
      go_msg_container = msg_container.
    ELSE.
      go_msg_container = cf_reca_message_list=>create( ).
    ENDIF.
  ENDMETHOD.

  METHOD get_msg_container.
    ro_msg_container = go_msg_container.
  ENDMETHOD.
ENDCLASS.

****************************************

TRY.
*    some code

    lo_message_list = cf_reca_message_list=>create( ).

*    fill message container in between

    RAISE EXCEPTION TYPE zcx_random_error
      EXPORTING
        msg_container = lo_message_list.

*    some other code

  CATCH zcx_random_error INTO DATA(lo_error).
*    add error messages to log
    lo_main_log->add_from_instance( lo_error->get_msg_container( ) ).
    lo_main_log->store( abap_false ).

*    show messages as popup
    CALL FUNCTION 'RECA_GUI_MSGLIST_POPUP'
      EXPORTING
        io_msglist = lo_error->get_msg_container( ).
ENDTRY.
```

Также работая в рамках программы или функционального модуля можно создать глобальный класс логгера, используя этот контейнер, для упрощения частого обращения к объекту для записи логов. Также можно построить вокруг контейнера класс-обертку, который будет инициализироваться с контейнером в статическом атрибуте где-то в начале программы и служить той же цели - упростить логирование, минимизировав количество boilerplate кода.

Стоит учесть, что в классе не реализованы некоторые тонкости работы с BAL, так что он не будет 100% заменой. Однако он с легкостью покрывает 99% повседневных задач по работе с сообщениями. Несмотря на то, что этот контейнер имеет довольно топорный интерфейс взаимодействия и то, что в нем смешаны несколько ответственностей - хранение сообщений, их обработка и логирование, он покрывает большинство задач, связанных с передачей сообщений внутри системы и всевозможном взаимодействии с ними.

## OData

Широкое применение концепция контейнера сообщений получила в рамках OData Gateway. SAP в рамках OData API предоставляет контейнер `/IWBEP/IF_MESSAGE_CONTAINER`, который используется для передачи сообщений через OData-канал клиенту OData-сервиса. Подробную информацию можно почитать [в справке по компоненту SAP_GWFND](https://help.sap.com/viewer/68bf513362174d54b58cddec28794093/7.5.9/en-US/01a226519eff236ee10000000a445394.html), а я приведу краткий обзору функциональности интерфейса и варианты работы с ним.
Интерфейс имплементирует класс `/iwbep/cl_mgw_msg_container`, в котором также есть статический factory-метод `get_mgw_msg_container`, через который возможно создать новый контейнер.

```abap
DATA:
  lo_msg_container  TYPE REF TO /iwbep/if_message_container.

lo_msg_container = /iwbep/cl_mgw_msg_container=>get_mgw_msg_container( ).
```

Этот контейнер используется в различных классах OData Gateway, в частности в качестве атрибута в базовых классах исключений `/iwbep/cx_mgw_busi_exception` и `/iwbep/cx_mgw_tech_exception`. Эти классы используются для возврата информации об ошибках для их последующего отображения на клиентской стороне. Я не буду подробно останавливаться на этих классах, скажу лишь, что от них удобно наследовать свои классы исключений при работе в OData сервисе и передавать с ними все сообщения об ошибках.

Теперь немного подробнее про сам контейнер.

Как и предыдущий, этот интерфейс имеет несколько методов для добавления сообщений.

```abap
lo_msg_container->add_message(
  iv_msg_type   = 'S'
  iv_msg_id     = 'RDA'
  iv_msg_number = 024
  iv_msg_v1     = 'test'
).

*message list from bapi
lo_msg_container->add_messages_from_bapi(
  it_bapi_messages = lt_bapi_result
).

*message list from BAL
lo_msg_container->add_messages_from_log(
  it_log_messages = lt_bal_messages
).

*message from another container
lo_msg_container->add_messages_from_container( lo_msg_container2 ).

* exception in CATCH-block 
lo_msg_container->add_message_from_exception( lo_error ).
```

Так как класс заточен на работу с OData, его методы содержат параметры, позволяющие задавать различные атрибуты ответа на запрос. Например, указывать, будет ли сообщение показываться в заголовке списка сообщений, или указывать название поля/компонента, спровоцировавшего ошибку. 

Обратите внимание, что метод  работает только для исключений, унаследованных от `/iwbep/cx_mgw_base_exception`. Остальные исключения можно обработать следующим образом - после поимки исключения выбросить новое исключение, унаследованное от `/iwbep/cx_mgw_base_exception`, передав текущее ему в качестве параметра, чтобы выше по стеку вызова уже работать с исключениями этого фреймворка. Сообщение из исходного исключения добавится в список ошибок при обработке исключения в самом фреймворке.

```abap
TRY.

    RAISE EXCEPTION TYPE zcx_random_error.
    
  CATCH cx_root INTO DATA(lo_some_error).
    RAISE EXCEPTION TYPE /iwbep/cx_mgw_tech_exception
      EXPORTING
        previous = lo_some_error.
ENDTRY.
```

Также в данном контейнере есть возможность добавлять сообщение из свободной текстовой переменной.

```abap
lo_msg_container->add_message_text_only(
  iv_msg_type = 'E'
  iv_msg_text = 'Some unexpected error!'
).
```

Возможно данный подход вполне оправдывает себя при отправке сообщений о технических ошибках через OData канал, но для логических (бизнес) ошибок всегда лучше использовать стандартные тексты, которые могут быть переведены на разные языки и легко найдены в коде при необходимости.

Как и предыдущий, этот контейнер обладает аналитическим функционалом.

```abap
IF lo_msg_container->get_worst_message_type( ) = 'E'.
*  error processing
ELSEIF lo_msg_container->get_worst_message_type( ) = 'W'.
*  not so bad
ELSE.
*  perfect!
ENDIF.
```

Также есть возможность получать сообщения в виде таблицы типа `/iwbep/t_message_container`.

Добавление сообщений при выбрасывании исключения может выглядеть следующим образом

```abap
lo_msg_container->add_message(
  iv_msg_type   = 'S'
  iv_msg_id     = 'RDA'
  iv_msg_number = 123
).

RAISE EXCEPTION TYPE /iwbep/cx_mgw_busi_exception
  EXPORTING
    message_container = lo_msg_container
    http_status_code  = /iwbep/cx_mgw_busi_exception=>gcs_http_status_codes-not_found.
```

Контейнер не содержит функционала по работе с логом, но в данном случае это и не предполагается, для этого есть специализированные решения, вроде предыдущего контейнера.

## BOPF

Фреймворк бизнес-объектов имеет свой контейнер сообщений, адаптированный под работу с древовидной структурой модели бизнес-объектов и особенностей их функционала.
Основным контейнером сообщений является `/BOBF/IF_FRW_MESSAGE`. Как и предыдущие рассмотренные примеры, контейнер создается через factory-метод соответствующего класса.

```abap
DATA:
  lo_msg_container TYPE REF TO /bobf/if_frw_message.

lo_msg_container = /bobf/cl_frw_message_factory=>create_container( ).
```

Метод может возвращать объекты разных классов в зависимости от контекста Transaction Manager’а, но для пользователя контейнера эта разница скрыта за интерфейсом.

Контейнер имеет возможность добавлять стандартные сообщения и сообщения из других контейнеров или из классов исключений.

```abap
lo_msg_container->add( lo_msg_container2 ).

lo_msg_container->add_message(
  is_msg = VALUE #(
    msgty = 'S'
    msgid = 'RDA'
    msgno = 024
    msgv1 = 'test'
  )
).

lo_msg_container->add_exception( lo_error ).
```

При добавлении сообщения можно указать специфичные для BOPF параметры - место возникновения сообщения: ключ ноды БО, ключ конкретной инстанции этой ноды, атрибут этой ноды, который содержит ошибочные данные.

Однако основной функционал этого контейнера реализован в рамках другого подхода. В BOPF используются объектно-ориентированные сообщения. В самом контейнере есть два метода по добавлению объектно-ориентированных сообщений - простое добавление сообщения и добавления сообщения с явным указанием места возникновения сообщения (location). Намного интереснее поговорить про сами классы сообщений, но об этом чуть позже.

Как и остальные контейнеры, `/BOBF/IF_FRW_MESSAGE` имеет методы для получения списка сообщений - возможно либо получить все сообщения таблицей, либо задать определенный фильтр.

```abap
DATA:
  lt_message_list      TYPE /bobf/cm_frw=>tt_frw,
  lt_detailed_msg_list TYPE   /bobf/t_frw_message_k.

*get all messages as object list
lo_msg_container->get(
  IMPORTING
    et_message = lt_message_list
).

*get error messages as detailed list
lo_msg_container->get_messages(
  EXPORTING
    iv_severity = /bobf/cm_frw=>co_severity_success
  IMPORTING
    et_message  = lt_detailed_msg_list
).
```

Также есть возможность проверить наличие сообщений с ошибками. Дополнительно можно указать, включать ли в проверку сообщения из actions или determinations бизнес-объектов. По-умолчанию проверяются все.

```abap
IF lo_msg_container->check( ).
*  error processing
ENDIF.
```

На этом основной функционал контейнера заканчивается. Куда интереснее подробнее рассмотреть концепцию сообщений,  реализованную в BOPF.

### Объектно-ориентированные сообщения, используемые в BOPF.

Как уже было сказано выше, бизнес-объекты используют для обработки сообщений объекты, наследующиеся от абстрактного класса `/BOBF/CM_FRW`. Технически он наследует класс `CX_DYNAMIC_CHECK`, то есть представляет собой класс исключений. Технически. [(Подробнее)](https://help.sap.com/viewer/cc0c305d2fab47bd808adcad3ca7ee9d/1709%20002/en-US/e82b394740b047ff8b86bb628f7a1ef2.html)

Смысл объектной обертки заключается в переходе от мета-сущности сообщений к объектам, которые удобно использовать в рамках программы (если, конечно, вы не из староверов, сидящих на процедурной парадигме). Такие сообщения довольно удобно создавать и использовать. Например, класс `/BOBF/CM_FRW_SYMSG`, который имеет интерфейс для добавления сообщения в привычном виде.

```abap
DATA(lo_message) = NEW /bobf/cm_frw_symsg(
  textid = VALUE #(
    msgid = 'RDA'
    msgno = 024
    attr1 = 'test'
  )
  severity = 'E'
  )
).
```

Их удобно добавлять в соответствующий контейнер.

```abap
lo_msg_container->add_cm( lo_message ).

lo_msg_container->add_cm(
  NEW /bobf/cm_frw_symsg(
    textid = VALUE #(
      msgid = 'RDA'
      msgno = 015
      attr1 = 'one'
      attr2 = 'two'
    )
    severity = 'S'
  )
).

MESSAGE e024(rda) WITH 'example' INTO DATA(lv_dummy).
lo_msg_container->add_cm(
  NEW /bobf/cm_frw_symsg(
    textid = VALUE #(
      msgid = sy-msgid
      msgno = sy-msgno
      attr1 = sy-msgv1
    )
    severity = sy-msgty
  )
).
```

Можно добавлять текст сообщений из строковой переменной (хоть я и считаю это и не очень хорошей практикой).

```abap
lo_msg_container->add_cm(
  NEW /bobf/cm_frw_symsg(
    message_text = 'some custom text'
  )
).
```

Также сообщения можно использовать как и обычные классы исключений, а при обработке на более высоком уровне добавить в контейнер. Особенно это удобно для того, чтобы обернуть сообщения от вызовов ФМ, используя соответствующий синтаксис 7.5 (не забудьте про интерфейс `IF_T100_DYN_MSG`).

```abap
TRY.

*    some coding

    RAISE EXCEPTION TYPE zcm_hello
      MESSAGE
      ID sy-msgid
      TYPE sy-msgty
      NUMBER sy-msgno
      WITH sy-msgv1 sy-msgv2 sy-msgv3 sy-msgv4.

*    more coding

    RAISE EXCEPTION TYPE zcm_hello
      EXPORTING
        textid   = zcm_hello=>some_error
        severity = zcm_hello=>co_severity_error.

*    some coding after all

  CATCH /bobf/cm_frw INTO DATA(lo_message).
    lo_msg_container->add_cm( lo_message ).

  CATCH cx_root INTO DATA(lo_error).
    MESSAGE lo_error TYPE 'X'.

ENDTRY.
```

Так как эти сообщения имплементируют интерфейс `IF_MESSAGE`, их можно использовать для отображения на экране при помощи оператора MESSAGE (хотя, если вы работаете в рамках BOPF, скорее всего интерфейс реализован на WebDynpro или Fiori, нежели на SAPGUI).

При создании объекта сообщения можно указать дополнительные параметры - как уже известное нам место возникновения сообщения в BOPF, так и другие - SEVERITY - тип сообщения, SYMPTOM - описывает причину возникновения ошибки, LIFETIME - описывает категорию сообщения (временные или постоянные: первые просто возвращаются пользователю и забываются, вторые сохраняются вместе с моделью (релевангны только для временных (draft) БО и будут появляться на последующих шагах обработки, пока проблема не будет устранена). 

```abap
lo_msg_container->add_cm(
  NEW /bobf/cm_frw_symsg(
    textid   = ls_textid
    severity = /bobf/cm_frw=>co_severity_warning
    symptom  = /bobf/if_frw_message_symptoms=>co_bo_inconsistency
    lifetime = /bobf/cm_frw=>co_lifetime_transition
  )
).
```

Несмотря на простоту и явность интерфейса объектных сообщений, он достаточно громоздок. Поэтому для упрощения рутинного кода в модулях, активно использующих BOPF, реализованы утилитарные классы для выполнения повторяющихся действий с контейнерами и сообщениями. Например в TM это `/SCMTMS/CL_MSG_HELPER`, а в EHS это `CL_EHFND_FW_APPL_LOG_HELPER`. Наверняка подобные есть и в других модулях, реализованных на бизнес-объектах - SLC, QIM, MOC и т.д. - найдите их самостоятельно. А если вы пришли на проект, где на бизнес-объектах реализована кастомерская логика с нуля (как на ISM-PrIMa в Deutsche Bahn), вы наверняка найдете с любовью написанный до вас Z-хелпер.

Для классов сообщений действует особый нейминг - \*cm\*, т.е. вы можете называть свои классы z\*cm\* или y\*cm\* (или /\*/cm\*, если вы работаете в вендорском пространстве имен). Семантически они ничем не отличаются от других классов исключений, и так же, как обычные классы исключений, могут иметь список сообщений, который можно редактировать через конструктор в se24 или вручную. Можно, например, добавить все сообщения какого-то модуля в один такой класс, а при создании сообщения передавать уже константу в textid - это сделает код более чистым, а сами сообщения можно будет найти через where-used list (работает только для глобальных классов).

```abap
CLASS zcm_hello DEFINITION
  PUBLIC
  INHERITING FROM /bobf/cm_frw
  CREATE PUBLIC .

  PUBLIC SECTION.

    INTERFACES if_t100_dyn_msg .

    CONSTANTS:
      BEGIN OF some_error,
        msgid TYPE symsgid VALUE 'RDA',
        msgno TYPE symsgno VALUE '024',
        attr1 TYPE scx_attrname VALUE '',
        attr2 TYPE scx_attrname VALUE '',
        attr3 TYPE scx_attrname VALUE '',
        attr4 TYPE scx_attrname VALUE '',
      END OF some_error .

    METHODS constructor
      IMPORTING
        !textid                  LIKE if_t100_message=>t100key OPTIONAL
        !previous                LIKE previous OPTIONAL
        !severity                TYPE ty_message_severity OPTIONAL
        !symptom                 TYPE ty_message_symptom OPTIONAL
        !lifetime                TYPE ty_message_lifetime DEFAULT co_lifetime_transition
        !ms_origin_location      TYPE /bobf/s_frw_location OPTIONAL
        !mt_environment_location TYPE /bobf/t_frw_location OPTIONAL
        !mv_act_key              TYPE /bobf/act_key OPTIONAL
        !mv_assoc_key            TYPE /bobf/obm_assoc_key OPTIONAL
        !mv_bopf_location        TYPE /bobf/conf_key OPTIONAL
        !mv_det_key              TYPE /bobf/det_key OPTIONAL
        !mv_query_key            TYPE /bobf/obm_query_key OPTIONAL
        !mv_val_key              TYPE /bobf/val_key OPTIONAL .
  PROTECTED SECTION.
  PRIVATE SECTION.
ENDCLASS.

CLASS zcm_hello IMPLEMENTATION.
  METHOD constructor.
    CALL METHOD super->constructor
      EXPORTING
        previous                = previous
        severity                = severity
        symptom                 = symptom
        lifetime                = lifetime
        ms_origin_location      = ms_origin_location
        mt_environment_location = mt_environment_location
        mv_act_key              = mv_act_key
        mv_assoc_key            = mv_assoc_key
        mv_bopf_location        = mv_bopf_location
        mv_det_key              = mv_det_key
        mv_query_key            = mv_query_key
        mv_val_key              = mv_val_key.
    CLEAR me->textid.
    IF textid IS INITIAL.
      if_t100_message~t100key = if_t100_message=>default_textid.
    ELSE.
      if_t100_message~t100key = textid.
    ENDIF.
  ENDMETHOD.
ENDCLASS.
```
*Сгенерированный класс*

Однако, несмотря на красоту описанного подхода, пока что он находит применение только внутри BOPF, в котором в принципе существует “своя атмосфера”. Поэтому для интеграции с другими частями SAP необходимо как-то преобразовать объектные сообщения в сообщения старой школы. Обычно такой функционал уже реализован в классах-хелперах модуля.

Несмотря на некоторые накладные расходы, сама концепция очень элегантна и позволяет работать с сообщениями в удобном формате, сочетающимся с современным стремлением ABAP стать нормальным языком программирования.

## Реализация в разных компонентах

Как мы все знаем, работники SAP никогда не общаются друг с другом, ходят в бар в разное время, а за обмен информацией между проектами можно положить учетку на стол и до конца жизни писать только в Z\*. По крайней мере этим можно было бы объяснить такое огромное количество разнообразных реализаций одного и того же, как в различных продуктах этой компании. Каждый модуль, каждый фреймворк и технология зачастую имеют собственные реализации одних и тех же базовых либо более специфических вещей, которые не совместимы друг с другом.

Не обошла стороной эта тенденция и контейнеры сообщений. Свои контейнеры есть у различных модулей и фреймворков. Вот несколько тех, которые я встречал в работе:

- `CL_LOG_PPF` - контейнер PPF
- `CL_EHFND_FW_LOGGER` - контейнер ENA (объектная обертка над BOPF) в EHS

Есть и другие, вы можете встретить их при работе с разными частями SAP. Но суть будет везде примерно одинакова, разве что адаптирована под контекст выполняемых задач.

## Что же выбрать?

В итоге мы имеем дюжину контейнеров сообщений с различным функционалом. Какой же лучше? Какой использовать в повседневной разработке?

Скорее всего, на этот вопрос нельзя дать однозначного ответа. Каждый хорош по-своему, поэтому используйте тот, который близок к контексту вашей разработки. Если вы разрабатываете OData-сервис, логично будет построить обмен сообщениями в рамках контейнера `/IWBEP/IF_MESSAGE_CONTAINER` и классов исключений, используемых в фреймворке. Аналогично и для других фреймворков или модулей - BOPF, PPF и т.д. В других случаях я советую использовать `IF_RECA_MESSAGE_LIST`, как наиболее универсальный. Если же в вашем модуле его нет (например в EWM - нет), поищите какую-то местную реализацию, скорее всего она там есть, если модуль моложе пятнадцати лет. Если и такого нет, воспользуйтесь каким-нибудь готовым Z-решением, благо их есть несколько довольно хороших. В любом случае такая обертка будет гораздо удобнее в работе, чем таскание туда-сюда таблиц вроде BAPIRET. 

Стремитесь к чистому объектно-ориентированному коду с минимумом неявных операций, коими являются многие операции с сообщениями, вроде классических исключений. Контейнеры сообщений помогут решить эту задачу элегантным и удобным способом, а также помогут использовать информацию из списка сообщений легким для разработчика путем.