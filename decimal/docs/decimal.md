# decimal — денежная арифметика для Yukino

Точные десятичные числа без float. `0.1 + 0.2` — это ровно `0.3`.
19 значащих цифр, до 9 знаков после точки, режимы округления включая
banker's rounding.

```yukino
load decimal: Dec, Round

set price = dec_parse("19.99")
set qty = dec_from_int(3)
set total = dec_mul(price, qty)          // 59.97
set vat = dec_mul(total, dec_parse("0.20")).quantize(2, Round::HalfEven)
print((total + vat).to_text())           // 71.96
```

## Установка

Пакет лежит в `yukino-lib/` и подключается как обычная зависимость:

```toml
# yuki.toml вашего проекта
[dependencies]
decimal = { path = "../yukino-decimal/yukino-lib" }
```

```yukino
load decimal: Dec, Round, dec_parse, dec_add, dec_mul
```

## Тип

`Dec` — класс с двумя полями: `coef: int64` и `scale: int64`.
Значение — `coef × 10^(−scale)`.

| | |
|---|---|
| значащих цифр | до 19 |
| масштаб | 0 … 9 |
| максимум | 9223372036854775807 (при `scale = 0`) |
| минимум по модулю | 0.000000001 |

Масштаб — часть значения: `1.5` и `1.50` равны, но печатаются по-разному.
`normalize()` убирает хвостовые нули.

## Создание

| функция | что делает |
|---|---|
| `dec_parse(s) -> Dec` | разбирает литерал, паникует `DecError` |
| `dec_try_parse(s) -> Dec?` | то же, но `null` вместо паники |
| `dec_from_int(v) -> Dec` | целое → `Dec` с масштабом 0 |
| `dec_raw(coef, scale) -> Dec` | из коэффициента и масштаба (тесты, сериализация) |

```yukino
dec_parse("19.99")      // (1999, 2)
dec_parse("1.5e3")      // (1500, 0)
dec_parse("1.2300")     // (12300, 4) — масштаб сохраняется
dec_parse("1,5")        // паника: ERR_PARSE
```

## Арифметика

| функция | семантика |
|---|---|
| `dec_add(a, b)` | точная сумма, `ERR_OVERFLOW` если не влезает |
| `dec_sub(a, b)` | точная разность |
| `dec_add_fit(a, b, mode)` | сумма с подгонкой масштаба |
| `dec_sub_fit(a, b, mode)` | то же для разности |
| `dec_mul(a, b)` | произведение, `HalfEven` для подгонки |
| `dec_mul_with(a, b, mode)` | произведение с явным режимом |
| `dec_mul_exact(a, b)` | произведение без подгонки, `ERR_PRECISION` если `scale > 9` |
| `dec_div(a, b, mode)` | деление, масштаб выбирается автоматически |
| `dec_quantize(d, scale, mode)` | привести к масштабу |
| `dec_compare(a, b)` | `−1 / 0 / 1`, числовое сравнение |

Точные операции **никогда** не округляют молча: если результат не
помещается в `int64`, будет `ERR_OVERFLOW`. Хотите округление — берите
`*_fit` или `quantize` и указывайте режим явно.

## Методы

```yukino
d.to_text()        // "19.99"
d.normalize()      // 1.2300 -> 1.23
d.quantize(2, Round::HalfEven)
d.negate()         // -d
d.abs_value()      // |d|
d.sign()           // -1 / 0 / 1
d.is_zero()
d.compare(other)   // -1 / 0 / 1
d.coef_value()     // коэффициент
d.scale_value()    // масштаб
```

Операторы (только для `Dec`, деление — `HalfEven`):

```yukino
a + b
a - b
a * b
a / b
a == b      // числовое: 1.5 == 1.50 → true
a < b
a > b
```

## Округление

`Round::Down`, `Up`, `Ceiling`, `Floor`, `HalfUp`, `HalfDown`, `HalfEven`.

```yukino
dec_quantize(dec_parse("2.5"), 0, Round::HalfEven).to_text()   // "2"
dec_quantize(dec_parse("3.5"), 0, Round::HalfEven).to_text()   // "4"
dec_quantize(dec_parse("1.005"), 2, Round::HalfEven).to_text() // "1.00"
dec_quantize(dec_parse("1.005"), 2, Round::HalfUp).to_text()   // "1.01"
dec_quantize(dec_parse("19.66"), 0, Round::Floor).to_text()    // "19"
```

`HalfEven` — режим по умолчанию для денег: он не накапливает смещение
на длинных рядах округлений.

## Ошибки

Ошибка — это `panic(DecError { code, message })`:

```yukino
load decimal: DecError, dec_parse

set code = ""
sus {
    set v = dec_parse("1,5")
} catch e: DecError {
    rewrite code = e.code          // "ERR_PARSE"
}
```

| код | причина |
|---|---|
| `ERR_PARSE` | нарушена грамматика литерала |
| `ERR_PRECISION` | масштаб больше 9 |
| `ERR_OVERFLOW` | промежуточное значение не влезает в `int64` |
| `ERR_DIV_BY_ZERO` | деление на ноль |

## Границы v1

* нет 128-битного коэффициента: произведение двух больших чисел, суммы
  с сильным расхождением масштабов и часть делений паникуют `ERR_OVERFLOW`;
* масштаб ограничен 9 знаками;
* `NaN`, `±Inf`, контексты и флаги IEEE 754 не поддерживаются — это
  денежный тип, а не полная реализация decimal128.

Точный список неподдержанных кейсов печатает `python mirror_check.py vectors`.

## Тесты

```bash
yukino test          # 23 ручных теста + 920 сгенерированных из корпуса
```

`tests/corpus_test.ykn` генерируется скриптом `gen_ykn_tests.py` из
корпуса `vectors/` — правки в него вносить бессмысленно, они потеряются.
