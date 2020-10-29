# PHP: Comparison list

This mem by @Fell [PHP: isset、is_null、if($var)、empty 比較一覧表](https://qiita.com/Fell/items/c63d01eca2a70ba30b6c)

## Comparison list

||Value or element| Type || isset | is_null | empty | if($var) | | !empty |
|-|---------------|------|-|------|---------|-------|----------|-|--------|
|1| $var | undefined(null) | |false|error(true)|true|error(false)||false|
|2| $var = NULL | null | |false|true|true|false||false|
|3| $var = ""; | string | |true|false|true|false||false|
|4(*)| $var = 0; | int | |true|false|true|false||false|
|5(*)| $var = '0' | string | |true|false|true|false||false|
|6| $var = 1 | int | |true|false|false|true||true|
|7| $var = '1' | string | |true|false|false|true||true|
|8| $var = array() | array | |true|false|true|false||false|
|9| $var = array(1) | array | |true|false|false|true||true|

## Comment 1

`isset` and `empty`

`empty($var)` is `!isset($var) || $var == false`

`!empty($var)` is `isset($var) && $var == true`

- `!isset` is safer than `is_null`
- `!empty` is a double check of `null check` and `if($var)`.
- Be careful of `$var = 0` and `$var = "0"` in (*).

## Summary

- `isset` is a `null check` including `undefined`.
- `empty` is a check when there is `no value`, including `undefined`.
- `if($var)` is a check when there is `a value` `without` `undefined check`.
- `!empty` is a check when there is `a value` with `undefined check`.
- The judgment part in `(*)` is quite delicate, and it is judged that there is no numerical 0 or "0" in the character string.

## variable in class

### getter

| |value    | type      || isset | is_null | empty | if($var) | | !empty|
| |---------|-----------|-|-------|--------|-------|----------| |-------|
|1|	$var	   |未定義(null)||	false | true	|true	|false	   | | false |
|2|	$var = NULL| null|	|false|	true|true|	false|	|false|
|3|	$var = ""; |string | |false|false|	true|	false|	|false|
|4(*)|	$var = 0;  |int	|	|false|	false|	true|	false|	|false|
|5(*)|	$var = "0";|int	|	|false|	false|	true|	false|	|false|
|6|	$var = 1;  |int	|	|false|	false|	true|	true|	|false|
|7|	$var = "1";|int	|	|false|	false|	true|	true|	|false|
|8|	$var = array()|	array|	|false|	false|	true|	false|	|false|
|9(*)|	$var = array(1)| array|	 |false| false|	true|	true|	|false|

### comment2

- With getters, it's different `!is_null` and `if($var)` are likely to be same.
- `isset` doesn't work at all. Use `!is_null`.
- Since `empty` is also `an array with contents` and the behavior is strange, `if($var)` seems to be good.
- Again, be careful of `$var = 0`, `$var = "0"` and `$var = array(1)` in (*).

## Reference

[PHP](https://www.php.net/manual/ja/types.comparisons.php)
