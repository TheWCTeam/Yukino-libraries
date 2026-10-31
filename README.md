# Yukino-libraries

Библиотеки для [Yukino](https://github.com/yukitsuri/yukino) — компилируемого
языка с регионами и affine ownership. Каждая библиотека — отдельный пакет
(`lib.toml` + `src/`), независимый от остальных.

## Пакеты

| пакет | что делает | тесты |
|---|---|---|
| [`decimal/`](decimal/) | денежная десятичная арифметика без float: 19 значащих цифр, 7 режимов округления, banker's rounding | 943 |

## Структура

```
Yukino-libraries/
├── decimal/
│   ├── lib.toml          манифест пакета (name = "decimal")
│   ├── src/lib.ykn       реэкспорт публичного API
│   ├── src/decimal.ykn   реализация, 625 строк
│   ├── tests/            23 ручных теста + 920 из корпуса
│   ├── examples/money.ykn
│   └── docs/decimal.md
└── README.md
```

## Подключение

```toml
# yuki.toml проекта
[dependencies]
decimal = { path = "../Yukino-libraries/decimal" }
```

```yukino
load decimal: Dec, Round, dec_parse, dec_mul, dec_from_int

set price = dec_parse("19.99")
set total = dec_mul(price, dec_from_int(3))
print(total.to_text())          // 59.97
```

## Тесты

```bash
cd decimal && yukino test
```

## Лицензия

Apache-2.0, как и сам Yukino. Подробности по `decimal` — в
[`decimal/docs/decimal.md`](decimal/docs/decimal.md).
