# Baseline для задачи "Расстановка товаров по полкам (X5 Retail Hero)"

Ссылка на задачу: https://retailhero.ai/c/shelf_allocation/overview

## Краткое описание:
* baseline, использующий динамическое программирование (реализация на Python и C++: [baseline.py](py/baseline.py), [baseline.cpp](cpp/baseline.cpp))
* быстрое вычисление метрики за O(h^3 * w) (реализация на Python и C++: [metric.py](py/metric.py), [baseline.cpp](cpp/baseline.cpp#L53-L141)(функции внутри))
* генерация тестов для локальной оценки качества [generate_tests.py](py/generate_tests.py)

## Как отправить

* baseline.py - файл с решением на Python, который можно отправить без докеров и регистрации;
* baseline.cpp - файл с решением на C++, который можно отправить без докеров и регистрации;

Результат: 48.6*10^6, на момент публикации 2-е место в лидерборде.


## Описание алгоритма
### Основная идея
**Горизонтальное заполнение** полки прямоугольниками-категориями, **оптимизирующеее размеры прямоугольников**, с использованием динамического программирования.

Пример заполнения (номерами обозначены категории):

```
11233455556
1123345555.
..2..4.....
..2........
```

при этом **внутри категорий используется жадное заполнение товарами**:
* выбираем в соответствии с размером прямоугольника топ наиболее прибыльных товаров категории;
* группируем товары по брендам и заполняем прямоугольник по строкам;

Пример заполнения прямоугольника размером 3х3 если всего в категории 4 бренда, в каждом из которых по 4 товара с прибыльностями c = 1,2,3,4:

```
brand   c
 111   432
 223   434
 344   343
```

### Логика динамического программирования

Пусть `c[i][j]` - максимальная оценка score, которую можно получить, заполнив полку размером ("h строк" на "j столбцов"), если использовать категории с номерами от 1 до i.

Тогда формула для динамики вперед выглядит так:

`c[i + 1][j + w_cat] = max(c[i + 1][j + w_cat], c[i][j] + cat_score[i][h_cat][w_cat])`, где
* размеры прямоугольника (h_cat, w_cat) для категории (i + 1) перебираются по всем допустимым значениям;
* `cat_score[i][h_cat][w_cat]` - оценка score для жадного алгоритма выбора товаров внутри категорий, описанного выше;

Для дальнейшего восстановления ответа в динамике запоминаются лучшие размеры прямоугольников `c_best[i][j]` - лучший прямоугольник (w_cat, h_cat) для i-й категории.
И за один проход после основного цикла восстанавливаем оптимальное решение.

## Логика быстрого расчета метрики за O(h^3 * w)

### Первое слагаемое метрики - score за категории:
Для каждой категории:
* находим мин/макс индексы строк и столбцов, на которых в итоговой матрице встречаются товары данной категории;
* проверяем, что все товары между найденными индексами принадлежат данной категории;
* добавляем к итоговому score "очки" за категорию в соответствии с формулой;

Асимптотика: O(h * w).

### Второе слагаемое метрики - score за бренды:

Для каждого товара (каждой ячейки матрицы) будем итеративно обновлять матрицу `score_matrix[i][j]` - 
максимальный score за бренды в соответствии с прибыльностью товара, расположенного в позиции (i, j) и формулой из описания.

Для этого перебираем индекс верхней строки: i от 1 до h и при фиксированном верхнем индексе будем итерироваться по индексу нижней строки j от i до h.

На каждой итерации, двигаясь по строке j слева направо будем поддерживать вектор состояния `state`, который позволит определять сплошные "кусочки", состоящие из товаров одного и того же бренда:
* при i=j (первая итерация цикла по нижнему индексу) `state[k] = a[i][k].brand`;
* далее на следующих итерациях `state[k]` обновляем по следующему правилу:
  * `state[k] = -1`, если `state[k] != a[i][k].brand`, т.е. если в добавленной строке j на позиции k бренд отличается от бренда на позиции k в векторе состояния (связь с брендом из предыдущих строк разрывается),
  * иначе `state[k]` не трогаем;

В процессе обновления вектора `state` если сплошной валидный кусочек прерывается или конец строки (т.е., если `state[k - 1] != -1 and (k == w or state[k] != state[k - 1])`), 
то можно обновить score для всех ячеек в подматрице с номерами строк от верхнего индекса до нижнего rows=i..j, и с номерами столбцов columns=left..k, 
где left - левый край последнего рассматриваемого сплошного кусочка, который также можно поддерживать во время итерации по строке.

Итоговый score - сумма элементов матрицы `score_matrix[i][j]`.

Асимптотика: O(h^3 * w).

