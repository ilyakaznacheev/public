#Страх и отвращение в ABAP CDS View 
 
Последний год я занимался созданием приложений в рамках [модели разработки для Fiori](https://help.sap.com/viewer/cc0c305d2fab47bd808adcad3ca7ee9d/7.52.3/en-US/3b77569ca8ee4226bdab4fcebd6f6ea6.html), которая почти полностью построена на BOPF и CDS. За это время у меня накопилось некоторое понимание особенностей и подводных камней, связанных с разработкой CDS View, которые стоит знать заранее и учитывать на этапе разработки архитектуры системы. Эти особенности оказывают существенное влияние на процесс разработки и поддержки по сравнению с классической ABAP разработкой. 
 
В этой заметке я изложу свои наблюдения по поводу некоторых узких мест или недостатков реализации приложений в рамках данной модели, с которыми я столкнулся лично. *Также хочу обратить внимание, что я пока не являюсь экспертом в этой технологии, поэтому могу где-то допустить неточности или не знать некоторых существующих инструментов, поэтому я приглашаю поправить меня в комментариях.* 
 
##Сложность отладки 
 
Одной из трудностей является невозможность отладки CDS View как таковых (что, в принципе, очевидно). То есть можно получить результат выборки, можно получить результат нижележащих выборок и примерно представить, как что сложилось, но когда у вас в одной CDS View выборка из 100+ различных таблиц, это может быть проблематично сделать. Чем больше логики реализуется внутри CDS View, тем больше будет возникать вопросов, почему одна из выборок вернула не то, что от нее ожидается, и возможности посмотреть в дебаггере по шагам что откуда берется у вас не будет. 
 
В итоге на практике, когда что-то выбирается не так, приходится сидеть и просматривать выборки различных CDS View в иерархии и руками фильтровать по ключевым полям на нижележащих выборках, что занимает очень много времени и требует излишней внимательности (а когда условие выборки накладывается на поле из ассоциации вроде `where _material._stock._location.locID <> ''` и фактически для восстановления реальной выборки, прошедшей под это условие, нужно руками пройти по трем ассоциациям для каждой строки и проверить условие там, хочется выбросить ноутбук в окно). В каких-то случаях это может упростить встроенная в ADT возможность проходить по ассоциациям внутри выборки, но она работает только для одной строки, а не для всей выборки.  
 
##Невозможность отладки или просмотра данных в продуктиве 
 
Другой особенностью является то, что в продуктивных системах возможности работы с CDS View обычно ограничены тем, что Eclipse нельзя подключить к системе, и, соответственно, вблизи посмотреть на выборку данных. Лучшее, что вы можете себе позволить - посмотреть выборку соответствующих CDS View ракурсов словаря через браузер данных вроде `se16n` (к которому, кстати, не всегда есть доступ в продуктивных системах). Поэтому, когда в продуктиве “что-то выбирается не так”, бывает очень проблематично, долго и болезненно докопаться до истины. 
 
##Неявные зависимости 
 
Еще одной особенностью является высокая связность CDS View. Изменения в одном месте могут совершенно неожиданным образом повлиять на выборку, которая как-то косвенно использует изменяемую. Это не такая уж большая проблема в небольших моделях, где все достаточно очевидно, но, когда в построение модели вовлечено больше, скажем, двадцати CDS View, сложно предусмотреть все возможные последствия одного изменения, как, например, дублирование или исчезновение каких-то данных из результирующих выборок.  
 
В теории это кажется незначительным и маловероятным (ведь "*мы должны предусмотреть влияние изменений на зависимые объекты*"), но на практике это намного сложнее, и встречаться с подобными сайд-эффектами все равно придется. Например, потому что в зависимости от различных данных в БД выборки будут вести себя по-разному, и определенное сочетание данных, которое приведет к определенным результатам, может не встретиться вам ни в разработческой, ни в тестовой системе, а уже в продуктиве. 
 
##Сложность покрытия тестами 
 
Еще одна особенность - юнит тестирование CDS View. Поскольку часть логики приложения опускается на уровень базы данных, эту логику, как и любую другую, необходимо покрывать тестами. SAP предоставляет такую возможность (с версии NW 7.5), однако такие тесты очень сложно актуализировать при внесении изменений в тестируемые выборки. Специфика CDS View такова, что даже незначительные изменения в логике выборки часто приводят к значительным изменениям в интерфейсе CDS View и структуре их иерархии. И такие изменения достаточно долго реализовывать в соответствующих юнит-тестах.  
 
Из-за этой особенностью покрывать тестами CDS View в процессе активной разработки - достаточно рискованная инвестиция времени, которая в конечном итоге может потребовать больше времени на актуализацию, чем сэкономит на проверке логики других выборок. 
 
##Ошибки БД 
 
Сюрпризом могут стать ограничения БД (в моем случае это HANA), которые не дадут широко развернуться при разработке логики на CDS View. Одно из них - ограничение на глубину запросов (количество join). Иными словами, в крупных и сложных моделях может получиться так, что построенная модель просто будет выдавать ошибку и не будет работать. Придется лезть в оптимизацию и переработку всей архитектуры. Иногда очень обидно бывает увидеть такое после активации изменений, над которыми работал весь день. 
 
Также иногда HANA просто возвращает случайные ошибки из-за каких-то внутренних проблем. К счастью, такое быстро проходит. Но если выборка очень большая, может и не пройти. 
 
Еще одна занимательная особенность - CDS View довольно долго активируются после переносов и иногда активируются с ошибками, которые, тем не менее, лечатся повторными переносами. В такие моменты система генерирует ошибку, которая говорит о том, что такой ракурс вообще не существует. Бывали случаи, что приходилось тратить целый день и десятки транспортов просто на то, чтобы перенести изменения так, чтобы все правильно активировалось в следующей системе, при том, что это был один и тот же набор изменений. 
 
Так или иначе, разрабатывая хоть сколько-то сложную логику на CDS View, будьте готовы внезапно встретиться с технической ошибкой БД (в рантайме), которую придется как-то обходить, что может оказаться болезненно. 
 
##Невозможность нормального профайлинга 
 
Также для CDS View не предусмотрено какого-либо адекватного профайлера, поэтому тюнинг производительности приходится выполнять “на глазок”. В рамках технологии есть множество косвенных указателей на сложность запроса, таких, как граф зависимостей, количество join, количество задействованных таблиц, категория размера и другие. Однако они дают лишь общее понимание того, как лучше работать с той или иной таблицей или насколько сложной получился тот или иной запрос, но не дают абсолютно никакого понимания реального положения дел, и не сильно помогают в процессе разработки или оптимизации понять, насколько хорош тот или иной подход с точки зрения производительности. 
 
Обычное профилирование ничего не дает, так как CDS View показывается как отдельная таблица (что, по сути, так и есть, иерархия множества CDS View компилируется в один огромный SQL-запрос), и никак нельзя посмотреть, что же конкретно дало особенную нагрузку на БД. 
 
Бывает даже такое, что чтобы проверить реальное влияние тех или иных изменений на код, приходится переносить изменения в другую систему, где достаточно данных для нагрузочного теста. 
 
##Сложность понимания влияния изменений на производительность 
 
Из предыдущего пункта вытекает другая проблема - неочевидность того, как те или иные изменения влияют на производительность. Можно добавить в одну из нескольких десятков CDS View новое поле из ассоциации какой-нибудь стандартной CDS View, а потом окажется, что при запросе на таблицу с полумиллионом записей это поле дало прирост времени выполнения 3000%. Печально обнаружить такое в продуктивной системе, когда приложение стало работать минуту вместо пары секунд. 
 
В процессе работы с технологией развивается некоторое “чутье” на узкие и опасные места, однако отсутствие возможности однозначно проверить производительность запроса и выловить узкие места добавляет много “веселья” в виде внезапно долгих запросов. 
 
##Негодность Open SQL для описания многих вещей 
 
Как уже было сказано в [другой моей заметке](http://www.kaznacheev.me/article/kak-nado-i-kak-ne-nado-ispolzovat-abap-cds-view/), CDS View не слишком то хорошо подходят для реализации бизнес-логики. Они вообще мало для чего подходят, кроме как для линейных выборок данных и агрегации некоторых полей. Почти вся логика, которая выходит за эти рамки, вызывает необходимость искать какие-то особые подходы для ее реализации в рамках доступных механизмов, что зачастую также негативно сказывается на производительности. Некоторые вещи вообще нельзя реализовать силами CDS View и приходится браться за CDS Table Functions (реализованных на [SQLScript](https://help.sap.com/viewer/de2486ee947e43e684d39702027f8a94/2.0.02/en-US) поверх AMDP), что, в свою очередь, зачастую очень плохо для производительности, так как Table Function является источником данных и возвращает целую таблицу, по которой уже делается выборка вышестоящей CDS View. 
 
Да и в целом Open SQL довольно ограничен в применении, и многие вещи все-таки лучше переложить на ABAP, если логика вашей программы чуть сложнее, чем табличный отчет. 
 
##Заключение 
 
Несмотря на все минусы, технология CDS View очень интересна и имеет множество достоинств в области моделирования данных. Старайтесь использовать ее именно для построения относительно небольших моделей с небольшим количеством внешних связей и не пихайте туда бизнес-логику (например условие “для этой БЕ здесь единичка, а для этой - двоечка, а других БЕ у нас вообще не может быть - так сказал клиент” - это уже бизнес-логика, а не модель). Не забывайте проверять производительность и старайтесь делать выборку из одной таблицы единожды, потому что HANA, похоже, не очень хорошо оптимизирует такие вещи. 

Судя по всему, скоро большинство продуктов SAP из ABAP линейки переедет на эту архитектурную парадигму, поэтому нужно заранее быть готовым и стараться писать модели, которые будут работать адекватно при разрастании продуктивных баз данных.