# Скрытие POI за зданиями. Полная версия

Наша карта уже достаточно давно рендерится при помощи технологий трехмерной графики. Сначала мы использовали эти технологии просто как очень быструю рисовалку двухмерных данных с небольшими исключениями в виде 3D-домов и моделей.

Приход в карту иммерсивных возможностей начал менять сложившееся положение вещей  --- моделей стало больше, они стали красивее и детальнее, появилось больше желания призумиться и рассматривать их. Наши картографические движки, заточенные на работу на масштабе города/отдельного района теперь должны научиться отображать ближние вьюпорты с более высокой детализацией. И тут возникла проблема:

![POI игнорируют 3D](problem.png)

Наши POI (иконки и надписи организаций и значимых мест) по-прежнему ощущают себя на двухмерной карте, игнорируя трехмерные геометрии. Это дизориентирует и создает визуальный мусор, ведь глядя на башни Сити меньше всего хочешь видеть иконки находящиеся  за 300-700 метров позади. Нужно было научить наши POI корректно рисоваться вместе с 3D объектами, скрываясь за ними при необходимости.

И мы начали экспериментировать.
## Решение при помощи буфера глубины

WebGL, как и все другие трехмерные API, предоставляет стандартную возможность для скрытия одних 3D объектов другими - [буфер глубины](https://ru.wikipedia.org/wiki/Z-буферизация). Если просто взять POI и включить им глубину, картинка будет следующая:

![Отрисовка POI с просто включенным буфером глубины](poi_depth_buffer.png)

Да, иконки начинают скрываться зданиями, однако вовсе не таким способом, каким ожидаешь - видно, что иногда здание отъедает кусок POI или часть надписи, делая ее нечитабельной.

Проблема в том, что POI не совсем "честный" 3D объект. Она сохраняет свою позицию на трехмерной сцене, но в то же время не подчиняется перспективе  - не уменьшается при удалении от камеры, не поворачивается вместе со сценой, оставаясь всегда лицом к пользователю. Это очевидное с точки зрение UX решение делает ее маргиналом в трехмерном пространстве. Использование буфера глубины напрямую нам не подходит.

Поэтому мы стали продумывать альтернативные способы решения.
## Аналитический подход

Аналитический подход предполагает расчет скрытия на CPU при помощи алгоритмов [бросания луча](https://ru.wikipedia.org/wiki/Ray_casting).

![Бросание луча](ray_casting.png)

Из камеры по направлению к каждой POI кидается луч, для которого проверяется пересечение с каждым 3D объектом.

У нас на карте запросто может быть несколько тысяч POI и несколько тысяч 3D объектов на экране, что в итоге дает миллионы итераций. А проверку пересечения нужно проводить на каждый кадр, то есть максимум за 10-15мс. Не проходит по производительности.
## Отдельная сцена скрытия POI

В этом подходе мы сгружаем вычисления на GPU. Для этого во вспомогательном [фреймбуфере](https://ru.wikipedia.org/wiki/Кадровый_буфер)  отрисовываем POI вместе с 3D объектами с использованием буфера глубины и пользуемся данными из этого фреймбуфера в шейдерах, при  отрисовке основной сцены. Условно говоря, если под центром POI в фреймбуфере красный цвет, то считаем его видимым, если красного цвета нет, то скрытым. 

Вот так примерно это выглядит - тут фреймбуфер скрытия POI полупрозрачно наложен на основную сцену.

![Скрытие POI при помощи вспомогательного фреймбуфера](labels_scene.jpg)

Отрисовка отдельного фреймбуфера влечет за собой накладные расходы по производительности, впрочем не столь большие как при аналитическом подходе. К тому же их можно уменьшить, снизив разрешение этого фреймбуфера. На картинке этот фреймбуфер имеет в 2 раза меньшее разрешение, это видно в виде большой пикселизованности красных областей.

В результате мы практически достигли нужно результата, но остался один момент:

![Моргание POI](poi_flickering.mov){width=100%}

Теперь POI начали скрываться целиком и не обрезаться прилегающими зданиями, однако они стали неприятно мерцать.

Причина этого кроется в том, что при пересечении POI и 3D-объекта возникают пиксельные решетки, при проходе по которым центра POI (по которому мы судим о видимости) слишком часто меняются состояния

![Пример пиксельной решетки](pixel_facet.png)

Преимущество данного подхода - скорость работы обернулась для нас недостатком. 

Борьба с пиксельной решеткой впоследствии оказалась достаточно нетривиальным делом. Мы перепробовали множество вариантов, начиная от различных способов антиалиасинга, блюра, до варианта с двумя фреймбуферами и моушен-блюром.

![Моушен-блюр](motion_blur.mp4){width=100%}

На экране иконки в сцене скрытия при движении оставляли за собой красивые хвосты, но успеха эти попытки нам не принесли.
## Асинхронная сцена скрытия

Другой наш подход заключался в том, чтобы мы расчитывали вспомогательный фреймбуфер не каждый кадр, а гораздо реже - где-то раз в 100-150мс и использовать его результаты в JS.

Для этого нам нужно отрисовать вспомогательный фреймбуфер и скачать его обратно из GPU в память CPU. После этого можно заглядывать в пикселы так же как и в шейдерах, благо спроецировать центр POI в координаты экрана является делом достаточно несложным.

Данный подход был нами реализован и нам удалось победить мерцание POI. Однако редкая отрисовка и последующее скачивание вспомогательного буфера приводили к отложенным изменениям в расположении POI на экране -- они долго изменяли свою конфигурацию после отзума или поворота карты.

![Асинхронная сцена скрытия](async_hide.mov){width=100%}
## Продакт-ревью

Становилось понятно, что времени на фичу ушло уже достаточно, полноценного решения  нет, зато есть множество неидеальных. Собрав их в кучу мы пошли к нашим продактам. Как и нас, ни один из вариантов их полностью не устроил.

Зато они отсекли вариант с асинхронной сценой, что позволило сосредоточиться нам на решении со скрытием в шейдерах. Еще продакты подсказали нам, что больше всего мерцания наблюдается у POI которые принадлежат какому-либо зданию, кроме того удобно было бы не скрывать POI собственным зданием. 

## Вторая итерация: текстура глубины и интерполяция прозрачности

После нескольких попыток борьбы с мерцанием было решено подключить к проверке не только цвет, но и глубину, чтобы при отрисовке POI понимать насколько ее глубина больше или меньше глубины здания или модели. Для этого мы использовали расширение для webgl, позволяющее получить буфер глубины в виде текстуры из фреймбуфера. Эта текстура имеет следующий вид:

![Текстура глубины](depth_texture.png)

Как можно заметить, чем объект ближе к камере, тем его вид темнее, т.е. значение глубины в текстуре меньше. И наоборот: чем светлее объект, тем глубина его больше и он располагается дальше от камеры.

Теперь при отрисовке POI в главной сцене у нас есть возможность сравнивать значения глубины зданий из текстуры и глубины POI: если глубина POI меньше глубины здания, то POI показывается, и наоборот - не показывается. Но проблема мерцания POI никуда не уходит. Для ее решения мы ввели некоторое значение дельты глубины, в границах которого мы интерполируем прозрачность POI, что позволило визуально сгладить процесс скрытия. Другими словами, у POI появилось переходное полупрозрачное состояние, т.е. если глубина POI меньше глубины здания - POI показывается; если больше, то мы вычисляем разность глубин POI и здания; если разность меньше значения дельты, то интерполируем прозрачность, и получаем полупрозрачную POI; если разность больше, то POI полностью скрыта. Логика скрытия POI изображена на картинке ниже.

![Логика скрытия POI](poi_hiding_logic.jpg)

Однако остается еще одна нерешенная проблема: если POI расположена внутри здания, то она всегда будет скрываться.

![POI внутри здания](building_covers_own_poi.png)

Мы сделали предположение, что если POI находится внутри здания, то она ему принадлежит, т.е., например, организация расположена внутри этого здания. Для решения этой проблемы нам пригодился буфер цвета из фреймбуфера. В данные каждой POI был добавлен id здания, которому она принадлежит, и т.к. этот id совпадает с id объекта самого здания, то этому id мы присвоили уникальный цвет и в буфер цвета отрисовали POI и само здание одним цветом. С этого момента в логике скрытия POI появилась еще одна проверка на цвет. Т.е. теперь как бы мы ни повернули камеру, из-за совпадения цвета здания и POI здание перестало влиять на ее скрытие.

![Цвета POI и зданий](building_poi_colors.png)

В итоге весь процесс скрытия построен на двух проверках. Сначала проверяем совпадение цвета POI с цветом из буфера цвета. Если цвета совпали, то POI не скрывается, т.е. ей ничего не мешает для отображения: ни собственное, ни какое-либо другое здание. Если цвета не совпали, то проверяем уже глубину POI и интерполируем прозрачность.

## Итого 

Не хотелось бы использовать избитые формулировки, но к данной фиче как нельзя лучше подходит "Было трудно, но мы справились". 

![Итоговая работа скрытия](final.mov){width=100%}

Фича получилась и она на бою. 

Было необычно в том плане, что типично трехмерную проблему не получилось все порешать внутри нашей команды, это была настоящая кросскомандная работа.

Мерцание POI до конца мы победить не смогли и оно все равно изредка происходит в особо сложных геометрических ситуациях. Будем надеяться, что в будущем мы найдем силы полностью искоренить эту проблему. 

Зато теперь стало значительно проще понять, где именно находится POI, и можно рассматривать наши красивые 3D-модели без помех. А красота спасет мир.