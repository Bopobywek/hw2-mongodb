# hw2-mongodb

```
{
$expr: {
$eq: [
    { $size: "$variants" },
    {
    $size: {
        $setIntersection: {
        $map: {
            input: "$variants",
            in: { size: "$$this.size" }
        }
        }
    }
    }
]
}
}
```

Проверяем, что схема удовлетворяет выражению. Размер множества (уникальные элементы) должен быть размеру variants

$setIntersection performs set operation on arrays, treating arrays as sets. If an array contains duplicate entries, $setIntersection ignores the duplicate entries  

Как раз за счет того, что он игнорирует дубликаты, мы получаем в ответе массив-множестов  

$map: {
    input: <expression> -- входной массив
    as: <string> -- алиас для элемента массива
    in: <expression> -- преобразование для элменета массива, здесь элемент массива  - документ, выбираем из этого документа уникальное поле
}

