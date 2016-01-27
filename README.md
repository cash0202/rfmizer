# RFMizer

Содержание:

* [Системные требования](#Системные-требования)
* [Использование](#Использование)
* [Требования к исходным данным](#Требования-к-исходным-данным)
  * [Формат CSV-файла с информацией о заказах](#Формат-csv-файла-с-информацией-о-заказах)
  * [Состав полей CSV-файла с информацией о заказах](#Состав-полей-csv-файла-с-информацией-о-заказах)
  * [Описание кофигурационного файла](#Описание-кофигурационного-файла)
* [Описание алгоритма](#Описание-алгоритма)

## Системные требования

- Python 2.7
- Установленный пакет `pyyaml`

Для установки `pyyaml` используйте [pip](https://pip.pypa.io/en/stable/installing/):

```
sudo pip install --user pyyaml
```

или

```
pip install --user pyyaml
```


## Использование

```
python rfmizer.py [--log-level LOG_LEVEL] config-file input-file
```

Обязательные аргументы:

- `config-file` — конфигурационный файл в формате [Yaml](http://yaml.org/)
- `input-file` — файл с информацией о заказах в формате [CSV](https://tools.ietf.org/html/rfc4180)

Необязательные аргументы:

- `--log-level LOG_LEVEL` — минимальный уровень сообщений, выдаваемых в процессе исполнения; возможные значения:
  - `CRITICAL`;
  - `ERROR`;
  - `WARNING` (уровень сообщений по умолчанию);
  - `INFO`;
  - `DEBUG`. 
  
`python rfmizer.py -h` отображает подсказку по использованию:

```
usage: rfmizer.py [-h] [--log-level LOG_LEVEL] config-file input-file

positional arguments:
  config-file           configuration file
  input-file            input data file

optional arguments:
  -h, --help            show this help message and exit
  --log-level LOG_LEVEL
                        logging level, defaults to WARNING
```

Примеры:

```
python rfmizer.py config.yaml orders.csv
python rfmizer.py --log=INFO config.yaml orders.csv
python rfmizer.py -h
```


## Требования к исходным данным

### Формат CSV-файла с информацией о заказах

- Файл должен быть в кодировке UTF-8.
- В начале файла НЕ должно быть маркера последовательности байтов (BOM, byte order marker).
- В качестве символа перевода строки должен использоваться символ «перевод строки»,
  он же «LF», он же «\n», он же «0x0A», он же «U+000A».
- Поля в строках должны разделяться запятыми: «,».
- В качестве символа десятичной точки в числах должен использоваться символ «.» (точка)
  или символ «,» (запятая).
- В полях, содержащих двойные кавычки «"», все символы двойных кавычек доджны быть задвоены:
  везде вместо «"» надо вставить «""»
- Поля, содержащие запятую «,», двойные кавычки «"» или переводы строки, должны быть заключены 
  двойные кавычки «"».


### Состав полей CSV-файла с информацией о заказах

Обязательные поля:

1. `order_date` — дата заказа;
2. `user_id` — уникальный обезличенный идентификатор покупателя;
3. `order_value` — cумма заказа.

Если присутствуют другие поля, то они могут расцениваться как поля, содержащие значения
дополнительных измерений, например, код географичекого расположения покупателя и т.п. Чтобы такие
поля были приняты за дополнительные измерения, их необходимо перечислить в секции `input_columns`
конфигурационного файла.


### Описание кофигурационного файла

```yaml
input_columns:
  - order_date
  - user_id
  - order_value
  - geo_code
segments_count:
  recency: 5
  frequency: 5
  monetary: 5
rfmizer:
  look_back_period: 365
  output_columns:
    user_id: ga:dimension1
    recency: ga:dimension2
    frequency: ga:dimension3
    monetary: ga:dimension4
    geo: ga:dimension5
predictor:
  prediction_period: 182
output_path: .
```

| Раздел | Параметр | Значение |
|---|---|---|
| | `input_columns` | Массив названий полей, которые надо учитывать в файле с заказами. Поля `order_date`, `user_id` и `order_value` **обязательно** должны присутствовать в файле и **обязательно** должны быть перечислены в этом параметре. |
| `segments_count` | `recency` | Количество сегментов, на которые будут поделены покупатели по измерению «Recency of last purchase». |
| `segments_count` | `frequency` | Количество сегментов, на которые будут поделены покупатели по измерению «Frequency of purchases». |
| `segments_count` | `monetary` | Количество сегментов, на которые будут поделены покупатели по измерению «Monetary life-time value». |
| `rfmizer` | `look_back_period` | Размер периода, на котором осущетсвляется сегментирование покупаталей. Указывается в днях. |
| `rfmizer` | `output_columns` | Словарь соответсвия названий измерений названиям полей в результирующем файле, содержащем соответсвие идентификаторов покупателей номерам их сегментов по каждому измерению. |
| `predictor` | `prediction_period` | Размер периода, на котором осуществляется расчет ценности каждого сегмента покупателей. Указывается в днях. |
| | `output_path` | Путь к директории, в которой сохраняются результирующие файлы. |


## Описание алгоритма

1. Построчно читаем исходный файл с информацией о заказах:
  1. Пропускаем строки, в которых не удалось определить дату заказа.
  2. Для каждого идентификатора пользователя (`user_id`) собираем словарь (dictionary), в котором ключами являются даты заказов, а значениями — суммы заказов данного пользователя за данную дату. В дальнейшем считаем все заказы, сделанные в один и тот де день, одним заказом на общую сумму за день.
  3. Если из заказов, сделанных пользователем за день, нет ни одного заказа с ненулевой и непустой суммой, то сумма заказов за этот день считается равной `None`. Это делается для того, чтобы такие дни не учитывать при расчете средней суммы заказа (monetary), но учитывать при расчете частоты заказов (frequency) и давности последнего заказа (recency).
  4. Попутно запоминаем для каждого пользователя дату последнего заказа, а также дату последнего заказа в исходном файле без учета пользователя — эта последняя дата используется как _сегодняшняя_, т.е. как начальная точка отсчета по времени.
  5. Для каждого `user_id` также запоминаем номера сегментов пользовательских (не-RFM) измерений, если они присутствуют в исходном файле с заказами и соответствующим образом перечислены в конфигурационном файле.
2. На временном периоде от сегодняшней даты (см. п. 1.iv) до `look_back_period` дней назад рассчитываем номера сегментов по измерениям «Recency» (давность последнего заказа), «Frequency» (частота заказов) и «Monetary» (средняя сумма заказа):
  1. Сначала для каждого пользователя расчитываем абсолютные значения метрик по измерениям:
    * количество заказов за период;
    * количество дней, прошедших со дня последнего заказа до сегодняшней даты;
    * среднюю сумму заказа без учета нулевых заказов и заказов, для которых в исходном файле не указана сумма.
  2. Для пользователей, у которых нет заказов за период, но есть более старые заказы, устанавливаем все метрики в значение `'stale'` — «протухшие». Информацию о пользователях, у которых нет заказов за период, но есть более _новые_ заказы, удаляем из массива пользователей. На первом проходе этого не происходит, поскольку последняя дата периода совпадает с датой последнего заказа в исходном файле, но на втором проходе — при расчете ценности сегментов (см. п. 4 ниже) — это позволяет рассчитать среднюю сумму заказов без учета пользователей, у которых все заказы оказались будущими по отношению к рассматриваемому периоду.
  3. Сегментируем пользователей по каждому измерению:
    1. Пользователей со значением соответствующей метрики, равным `'stale'`, относим к нулевому сегменту.
    2. Пользователей со значением соответствующей метрики, равным `None`, относим к первому сегменту.
    3. Остальных пользователей упорядочиваем в порядке возрастания соответствующей метрики.
    4. Считаем минимальное количество пользователей в первом сегменте: делим общее количество упорядоченных пользователей на `segments_count` для данного измерения.
    5. Проходим массив упорядоченных пользователей от начала и помечаем пользователей, как относящися к первому сегменту до тех пор, пока не будут удовлетворедны два условия:
      * количество пользователей будет не меньше минимального;
      * значение метрики очередного пользователя будет отличаться от значения метрики предыдущего пользователя.
    6. Запоминаем значение метрики, при котором «закончились» пользователи, относящиеся к первому сегменту.
    7. Для следующих сегментов выполняем пункты 2.iii.d–2.iii.f на оставшихся пользователях.
3. Сохраняем в файл рассчитанные значения номеров сегментов для каждого `user_id`. Помимо расчитанных сегментов сохраняем также номера сегментов пользовательских (не-RFM) измерений, если они присутствовали в исходном файле с заказами и были соответствующим образом перечислены в конфигурационном файле. Пример файла с номерами сегментов для каждого пользователя:  
  `user_id,recency,frequency,monetary,geo`  
  `123,1,2,4,3`  
  `456,2,1,2,1`  
  `789,3,2,4,7`
4. Выполняем п. 2, приняв за точку отсчета дату, отстоящую на `prediction_period` дней назад от сегодняшней (см. п. 1.iv), т.е. проводим сегментацию пользователей по RFM-измерениям на более старом временном периоде от `prediction_period` дней назад до `prediction_period + look_back_period` дней назад. При этом используем границы сегментов по каждому измерению, рассчитанные на предыдущем проходе, т.е. на периоде от сегодняшней даты (см. п. 1.iv) до `look_back_period` дней назад.
5. Подсчитываем среднюю сумму заказов для всех пользователей на периоде от сегодняшней даты до `prediction_period` дней назад.
6. Подсчитываем среднюю сумму заказов для пользователей каждого сегмента по каждому измерению отдельно на периоде от сегодняшней даты до `prediction_period` дней назад. Расчет выполняется как для RFM-измерений, так и для пользовательских измерений.
7. Для каждого сегмента каждого измерения подсчитываем относительную ценность сегмента — отношение средней суммы заказов пользователей в сегменте к средней сумме заказов для всех пользователей.
8. Перебираем все возможные комбинации сегментов по всем измерениям и определяем ценность микросегментов, перемножая относительные ценности сегментов, определенные в п. 7. Под микросегментами подразумеваем пересечения сегментов разных измерений.
9. Сохраняем в файл информацию об относительно ценности каждого микросегмента. Пример файла со значениями  относительной ценности каждого сегмента:  
  `recency,frequency,monetary,geo,bid ratio`  
  `1,1,1,1,0.9`  
  `2,1,1,1,1.2`  
  `3,1,1,1,1.1`
