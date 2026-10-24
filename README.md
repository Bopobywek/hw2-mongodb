# ДЗ2

## Данные и запросы  
### Модель  
Заказ - отдельная сущность, которая включает в себя часть products, поэтому хранится отдельным документом.  
Позиция встроена в заказ, так как нам необходимо хранить небольшой снэпшот данных продукта (цена, количество) + размер (для быстрого получения, дерномализация для оптимизации).  

В позиции заказа цена представляет факт совершения покупки по цене на момент покупки. То есть в будущем цена товара может измениться, в совершенном заказе она меняться не должна.

## Отчет о продажах 
Расчет на примере товара "Футболка" (sku: A) на данных с 2026-01-01 до 2026-02-01
#### Шаг 1: match  
Ищем все данные, подходящие заказы под условие `status: paid` и `2026-01-01 <= createdAt < 2026-02-01`    
Подходят заказы:  
- `"_id": "sale-start"`
- `"_id": "sale-middle"`
- `"_id": "sale-last"`

Не подошли:  
- `"_id": "excluded-end"`: дата уже не подходит
- `"_id": "excluded-before"`: заказ сделан до нужного периода
- `"_id": "excluded-created"`: заказ не в нужном статусе
- `"_id": "excluded-cancelled"`: заказ не в нужном статусе

#### Шаг 2: unwind  
Превращаем items в плоскую структуру

#### Шаг 3: group  
Группировка по sku, расчитываем в группе qty как сумму количества проданных товаров и выручку как цену продажи на количество  

Так, для футболок:  
```
_id: A
qty: 2 + 1 = 3
revenue: 2 * 100 + 1 * 120 = 320
```

#### Шаг 4: sort  
Сортируем по выручке по убыванию, потом по возрастанию. 
Футболки оказываются на 2 месте

#### Шаг 5: lookup  
Теперь нам необходимо соединить получившиеся документы с документами в продуктах, чтобы вывести имя товара  

Соединяем по полю _id - в двух коллекциях у нас там sku товара

#### Шаг 6: unwind  
lookup все кладет результат джоина в массив, даже если элемент 1 или ничего нет. Поэтому нам нужно превратить документ в плоскую структуру. Используем unwind

#### Шаг 7: project  
Делаем проекцию, указываем ккаие поля попадут в результат  

Так, для футболок получаем 
{ qty: 3, revenue: Decimal128('320.00'), sku: 'A', name: 'Футболка' }

## Индекс и план выполнения

```
docker compose run --rm lab reset index
docker compose run --rm lab shell

hw2rs [primary] hw2> load("/work/solution.js")
true
hw2rs [primary] hw2> historyCursor(db, { userId: 2500, status: 'paid' }).explain('executionStats')
```

Получил следующий план:

```
{
  explainVersion: '1',
  queryPlanner: {
    namespace: 'hw2.orders',
    parsedQuery: {
      '$and': [ { status: { '$eq': 'paid' } }, { userId: { '$eq': 2500 } } ]
    },
    indexFilterSet: false,
    queryHash: 'DFD91C33',
    planCacheShapeHash: 'DFD91C33',
    planCacheKey: 'FB45820F',
    optimizationTimeMillis: 0,
    maxIndexedOrSolutionsReached: false,
    maxIndexedAndSolutionsReached: false,
    maxScansToExplodeReached: false,
    prunedSimilarIndexes: false,
    winningPlan: {
      isCached: false,
      stage: 'SORT',
      sortPattern: { createdAt: -1, _id: 1 },
      memLimit: 104857600,
      limitAmount: 20,
      type: 'simple',
      inputStage: {
        stage: 'COLLSCAN',
        filter: {
          '$and': [
            { status: { '$eq': 'paid' } },
            { userId: { '$eq': 2500 } }
          ]
        },
        direction: 'forward'
      }
    },
    rejectedPlans: []
  },
  executionStats: {
    executionSuccess: true,
    nReturned: 20,
    executionTimeMillis: 6,
    totalKeysExamined: 0,
    totalDocsExamined: 10000,
    executionStages: {
      isCached: false,
      stage: 'SORT',
      nReturned: 20,
      executionTimeMillisEstimate: 1,
      works: 10022,
      advanced: 20,
      needTime: 10001,
      needYield: 0,
      saveState: 0,
      restoreState: 0,
      isEOF: 1,
      sortPattern: { createdAt: -1, _id: 1 },
      memLimit: 104857600,
      limitAmount: 20,
      type: 'simple',
      totalDataSizeSorted: 4960,
      usedDisk: false,
      spills: 0,
      spilledDataStorageSize: 0,
      inputStage: {
        stage: 'COLLSCAN',
        filter: {
          '$and': [
            { status: { '$eq': 'paid' } },
            { userId: { '$eq': 2500 } }
          ]
        },
        nReturned: 1600,
        executionTimeMillisEstimate: 1,
        works: 10001,
        advanced: 1600,
        needTime: 8400,
        needYield: 0,
        saveState: 0,
        restoreState: 0,
        isEOF: 1,
        direction: 'forward',
        docsExamined: 10000
      }
    }
  },
  queryShapeHash: '635D4F00B666E61B713823BFECAABA5DFB3B4184E1C2F1FC6D7392224E8D6FD2',
  command: {
    find: 'orders',
    filter: { userId: 2500, status: 'paid' },
    sort: { createdAt: -1, _id: 1 },
    limit: 20,
    '$db': 'hw2'
  },
  serverInfo: {
    host: 'a5f00093e60e',
    port: 27017,
    version: '8.0.30',
    gitVersion: '360770fbc9b6a893748150d2ab2799e119b9fac9'
  },
  serverParameters: {
    internalQueryFacetBufferSizeBytes: 104857600,
    internalQueryFacetMaxOutputDocSizeBytes: 104857600,
    internalLookupStageIntermediateDocumentMaxSizeBytes: 104857600,
    internalDocumentSourceGroupMaxMemoryBytes: 104857600,
    internalQueryMaxBlockingSortMemoryUsageBytes: 104857600,
    internalQueryProhibitBlockingMergeOnMongoS: 0,
    internalQueryMaxAddToSetBytes: 104857600,
    internalDocumentSourceSetWindowFieldsMaxMemoryBytes: 104857600,
    internalQueryFrameworkControl: 'trySbeRestricted',
    internalQueryPlannerIgnoreIndexWithCollationForRegex: 1
  },
  ok: 1,
  '$clusterTime': {
    clusterTime: Timestamp({ t: 1792841373, i: 1 }),
    signature: {
      hash: Binary.createFromBase64('AAAAAAAAAAAAAAAAAAAAAAAAAAA=', 0),
      keyId: Long('0')
    }
  },
  operationTime: Timestamp({ t: 1792841373, i: 1 })
}
```

Добавим индекс 
```
globalThis.historyIndex = { userId: 1, status: 1, createdAt: -1, _id: 1 };
```

Индекс по правилу ESR  
- E = (userId: 1, status: 1)
- S = (createdAt: -1, _id: 1)

Получим полный план для запроса после добавления индекса

```
docker compose run --rm lab index
docker compose run --rm lab shell

hw2rs [primary] hw2> load("/work/solution.js")
true
hw2rs [primary] hw2> historyCursor(db, { userId: 2500, status: 'paid' }).explain('executionStats')
```

```
{
  explainVersion: '1',
  queryPlanner: {
    namespace: 'hw2.orders',
    parsedQuery: {
      '$and': [ { status: { '$eq': 'paid' } }, { userId: { '$eq': 2500 } } ]
    },
    indexFilterSet: false,
    queryHash: 'DFD91C33',
    planCacheShapeHash: 'DFD91C33',
    planCacheKey: 'F0D6D863',
    optimizationTimeMillis: 0,
    maxIndexedOrSolutionsReached: false,
    maxIndexedAndSolutionsReached: false,
    maxScansToExplodeReached: false,
    prunedSimilarIndexes: false,
    winningPlan: {
      isCached: false,
      stage: 'LIMIT',
      limitAmount: 20,
      inputStage: {
        stage: 'FETCH',
        inputStage: {
          stage: 'IXSCAN',
          keyPattern: { userId: 1, status: 1, createdAt: -1, _id: 1 },
          indexName: 'history_user_status_date_id',
          isMultiKey: false,
          multiKeyPaths: { userId: [], status: [], createdAt: [], _id: [] },
          isUnique: false,
          isSparse: false,
          isPartial: false,
          indexVersion: 2,
          direction: 'forward',
          indexBounds: {
            userId: [ '[2500, 2500]' ],
            status: [ '["paid", "paid"]' ],
            createdAt: [ '[MaxKey, MinKey]' ],
            _id: [ '[MinKey, MaxKey]' ]
          }
        }
      }
    },
    rejectedPlans: []
  },
  executionStats: {
    executionSuccess: true,
    nReturned: 20,
    executionTimeMillis: 0,
    totalKeysExamined: 20,
    totalDocsExamined: 20,
    executionStages: {
      isCached: false,
      stage: 'LIMIT',
      nReturned: 20,
      executionTimeMillisEstimate: 0,
      works: 21,
      advanced: 20,
      needTime: 0,
      needYield: 0,
      saveState: 0,
      restoreState: 0,
      isEOF: 1,
      limitAmount: 20,
      inputStage: {
        stage: 'FETCH',
        nReturned: 20,
        executionTimeMillisEstimate: 0,
        works: 20,
        advanced: 20,
        needTime: 0,
        needYield: 0,
        saveState: 0,
        restoreState: 0,
        isEOF: 0,
        docsExamined: 20,
        alreadyHasObj: 0,
        inputStage: {
          stage: 'IXSCAN',
          nReturned: 20,
          executionTimeMillisEstimate: 0,
          works: 20,
          advanced: 20,
          needTime: 0,
          needYield: 0,
          saveState: 0,
          restoreState: 0,
          isEOF: 0,
          keyPattern: { userId: 1, status: 1, createdAt: -1, _id: 1 },
          indexName: 'history_user_status_date_id',
          isMultiKey: false,
          multiKeyPaths: { userId: [], status: [], createdAt: [], _id: [] },
          isUnique: false,
          isSparse: false,
          isPartial: false,
          indexVersion: 2,
          direction: 'forward',
          indexBounds: {
            userId: [ '[2500, 2500]' ],
            status: [ '["paid", "paid"]' ],
            createdAt: [ '[MaxKey, MinKey]' ],
            _id: [ '[MinKey, MaxKey]' ]
          },
          keysExamined: 20,
          seeks: 1,
          dupsTested: 0,
          dupsDropped: 0
        }
      }
    }
  },
  queryShapeHash: '635D4F00B666E61B713823BFECAABA5DFB3B4184E1C2F1FC6D7392224E8D6FD2',
  command: {
    find: 'orders',
    filter: { userId: 2500, status: 'paid' },
    sort: { createdAt: -1, _id: 1 },
    limit: 20,
    '$db': 'hw2'
  },
  serverInfo: {
    host: 'a5f00093e60e',
    port: 27017,
    version: '8.0.30',
    gitVersion: '360770fbc9b6a893748150d2ab2799e119b9fac9'
  },
  serverParameters: {
    internalQueryFacetBufferSizeBytes: 104857600,
    internalQueryFacetMaxOutputDocSizeBytes: 104857600,
    internalLookupStageIntermediateDocumentMaxSizeBytes: 104857600,
    internalDocumentSourceGroupMaxMemoryBytes: 104857600,
    internalQueryMaxBlockingSortMemoryUsageBytes: 104857600,
    internalQueryProhibitBlockingMergeOnMongoS: 0,
    internalQueryMaxAddToSetBytes: 104857600,
    internalDocumentSourceSetWindowFieldsMaxMemoryBytes: 104857600,
    internalQueryFrameworkControl: 'trySbeRestricted',
    internalQueryPlannerIgnoreIndexWithCollationForRegex: 1
  },
  ok: 1,
  '$clusterTime': {
    clusterTime: Timestamp({ t: 1792841543, i: 1 }),
    signature: {
      hash: Binary.createFromBase64('AAAAAAAAAAAAAAAAAAAAAAAAAAA=', 0),
      keyId: Long('0')
    }
  },
  operationTime: Timestamp({ t: 1792841543, i: 1 })
}
```

Неизменность списка _id и его порядка подтверждается запуском сценария index
```
PASS same IDs and order
PASS matches independent fixture order
```

> hint не использовал

| Вариант       | nReturned | totalKeysExamined | totalDocsExamined | Есть отдельная SORT |
|---------------|-----------|-------------------|-------------------|---------------------|
| До индекса    | 20        | 0                 | 10000             | true                |
| После индекса | 20        | 20                | 20                | false               |


До появления индекса, стандартный индекс по _id не использовался - это можно сказать
по totalKeysExamined=0. Нужно было обойти все 10к записей, чтобы найти подходящие + нужна сортировка  

После появления индекса, уже видим, что он использовался totalKeysExamined != 0.
Обращения к документам есть, так как на выход отдается весь документ, а не только поля из индекса

**Издержка индекса** - память и обновление индекса

