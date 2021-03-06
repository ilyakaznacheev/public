# Использование CDS View как источник данных для SALV IDA 
 
В SAP NW версии 7.4 появилась технология под названием IDA. Ее суть заключается в том, что данные для отображения в ALV таблице выбираются не заранее в программе, а считываются напрямую во время отображения таблицы. Как и обычный SALV, она имеет огромный функционал, частью которого является использование CDS View в качестве источника данных. Взглянем на эту часть поближе. 
 
С моей точки зрения сам подход IDA довольно спорный с точки зрения архитектуры. В больших приложениях прямой доступ к данным из таблицы в ALV ломает разделение на уровни отображения, если вы используете что-то вроде MVP. Но для небольших приложений и с точки зрения производительности он достаточно неплох. В небольших отчетах, где ключевым элементом является таблица БД, и вся логика строится вокруг нее, использование IDA позволит сократить время на разработку. 

Как все знают, SALV не предоставляет инструментов для редактирования табличных данных. Есть определенные пути, позволяющие сделать SALV редактируемым, но я их рассматривать и описывать не буду, так как это выходит за рамки этой заметки. 

Объект IDA для SALV для CDS View отличается от обычного тем, что для его создания вызывается специальный фабричный метод. 

```abap 
DATA(lo_table) = cl_salv_gui_table_ida=>create_for_cds_view( 
 iv_cds_view_name = 'S_FLIGHTS' 
). 
``` 
 
Основное отличие IDA от обычного SALV заключается в том, что, по сути, это обертка над `SELECT`, параметры которого разработчик задает в ORM-подобном виде. В случае CDS View это параметры, передаваемые при выборке, и WHERE-условия. 

Параметры задаются при помощи метода `set_view_parameters`: 

```abap 
lo_table->set_view_parameters( 
 VALUE #( 
   ( 
     name = 'PARAM_NAME' 
     value = lv_param_value 
   ) 
). 
``` 

При задании WHERE-условий поддерживаются как range, так и обычные значения. Для задания условий используется специальный объект, который можно получить из IDA: 

```abap 
DATA(lo_conn_f) = lo_table->condition_factory( ). 
``` 

Этот объект дальше используется для последовательного задания WHERE-условия, и может выглядеть как-то так: 

```abap 
DATA(lo_condition) = lo_conn_f->in_range( 
 name      = 'CONNECTIONID' 
 t_ranges  = s_conn[] 
)->and(  
 other_condition = lo_conn_f->between( 
   name    = 'FLIGHTDATE' 
   low    = '20100101' 
   high    = '20200101' 
 ) 
)->or(  
 other_condition = lo_conn_f->not_covers_pattern( 
   name    = 'CARRIERID' 
   pattern = 'A%' 
 )->and(  
   other_condition = lo_conn_f->greater_or_equal( 
     name = 'PRICE' 
     value = 100 
   ) 
 ) 
). 
``` 
 
Этот код в `SELECT` запросе выглядел бы как: 

```abap 
 WHERE 
   connectionid  IN @s_conn 
AND flightdate    BETWEEN '20100101' AND '20200101' 
OR ( carrierid NOT LIKE 'A%' 
AND price >= 100 ). 
``` 

Потом эти условия устанавливаются IDA для ограничения выборки. 

```abap 
lo_table->set_select_options( io_condition  = lo_condition ). 
``` 

И наконец таблица выводится на экран. В данном случае я использую полноэкранный режим на экране по-умолчанию, но вы можете отобразить ее и в контейнере. 
 
```abap 
lo_table->fullscreen( )->display( ). 
``` 
 
Для того, чтобы обновить выборку после изменений в таблице, используйте метод `refresh( )`. Фактически это реализует паттерн MVC относительно модели, описываемой CDS View. 
 
Как и в обычном SALV, в IDA есть широкие возможности по настройке внешнего вида таблицы и работы с ее данными. Можно добавлять кнопки, выделять строки, обрабатывать двойной щелчек, устанавливать свои заголовки и так далее. Несмотря на то, что организация методов несколько отличается от обычного SALV, в целом функционал примерно такой же, поэтому я не буду подробно его описывать. 
 
 
Еще одна интересная особенность IDA для CDS View состоит в том, что в самом View через аннотации можно указать названия и всплывающие подсказки для табличных колонок. Выглядит это следующим образом: 
 
```sql 
@EndUserText.quickInfo: 'Оборот за выбранный промежуток' 
@EndUserText.label: 'Оборот' 
Turnover 
``` 
 
Что в итоге будет выглядеть следующим образом: 
 
![заголовок таблицы](https://habrastorage.org/webt/cw/wu/wy/cwwuwyt9g-sh4nyalztrahfz9tk.png) 
 
Эти тексты будут доступны для перевода через `SE63`, так что нет нужды волноваться по поводу локализации. Также это намного удобнее и быстрее, чем создавать отдельные дата-элементы или потрошить каталог полей в программе. 
  
Также при помощи CDS Access Control можно описать права доступа к данным CDS View, и эти права будут применены при чтении данных в IDA на основе присвоенных ролей. 
 
Обратите внимание, что факт проверки ролей в CDS Access Control не отображается в `SU53`, так как, по сути, CDS View выдает только данные, на которые права есть, и инцидента безопасности как такового не происходит. Однако это можно просмотреть через транзакцию `STAUTHTRACE`, так что не стоит беспокоиться, что данные пропадут бесследно. 
 
 
Полный пример CDS View и программы для ее отображения с различными условиями можно [посмотреть здесь](https://gist.github.com/ilyakaznacheev/34cfa7599ec4b5fc69a5a6344d51335f). 
 
 
По опыту могу сказать, что IDA очень хорошо справляется с задачей отображения таблиц без необходимости множественного редактирования строк. Технология включает большое количество различных возможностей по взаимодействию, как добавление кнопок, даблклики и т.п., что позволяет реализовать необходимый функционал по работе с таблицей в достаточной мере. При этом пользователю не нужно беспокоиться о выборке данных и их хранении в программе, а также об оптимизации для больших таблиц. Использование же CDS View в качестве источника данных позволяет как добавить описание столбцов и реализовать быструю проверку прав на чтение, так и сформировать именно ту модель данных, которая нужна для отображения.

*P.S.:*

На практике оказалось, что в некоторых версиях системы через аннотации меняется только короткий текст названий столбцов, если же у нижележащего элемента данных есть свои тексты, то они отображаются в ALV при средней и большой ширине столбца. А происходит это из-за ошибки в стандартном методе `create_field_descr_4_entity` класса `cl_salv_ida_structdescr`. Тексты из элемента данных должны очищаться, если в CDS View есть аннотация с текстом, однако этого не происходит. Не понятно, когда SAP исправит эту ошибку, пока можете воспользоваться [таким костылем](https://gist.github.com/ilyakaznacheev/32bb3e2a1b8ee2a46e826ceccc27c656).