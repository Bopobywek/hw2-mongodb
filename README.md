# ДЗ2

## Окружение 

Использовал подготовленные lab.js и compose.yaml

## Данные и запросы  
Для проведения автоматической проверки использовал  
```
docker compose run --rm lab data
```

### Модель  
Заказ - отдельная сущность, которая включает в себя часть products, поэтому хранится отдельным документом.  
Позиция встроена в заказ, так как нам необходимо хранить небольшой снэпшот данных продукта (цена, количество) + размер (для быстрого получения, дерномализация для оптимизации).  

В позиции заказа цена представляет факт совершения покупки по цене на момент покупки. То есть в будущем цена товара может измениться, в совершенном заказе она меняться не должна.

## Отчет о продажах   
Для проведения автоматической проверки использовал  
```
docker compose run --rm lab data
```

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

#### Шаг 5: limit  
Топ-5 товаров

#### Шаг 6: lookup  
Теперь нам необходимо соединить получившиеся документы с документами в продуктах, чтобы вывести имя товара  

Соединяем по полю _id - в двух коллекциях у нас там sku товара

#### Шаг 7: unwind  
lookup все кладет результат джоина в массив, даже если элемент 1 или ничего нет. Поэтому нам нужно превратить документ в плоскую структуру. Используем unwind

#### Шаг 8: project  
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

**Издержка индекса**: индекс занимает дополнительное место в памяти, помимо этого его необходимо обновлять с операциями изменения

## Оформление заказа
Для проведения автоматической проверки использовал  
```
docker compose run --rm lab purchase
```
Пошагово вручную проверял следующим образом:  
```
docker compose run --rm lab reset
docker compose run --rm lab shell

load('/work/solution.js')

globalThis.HWError = (code, message = code) => Object.assign(new Error(message), { code })
globalThis.transaction = async body => {
  const session = db.getMongo().startSession({ causalConsistency: true })
  try {
    session.startTransaction({
      readConcern: { level: 'snapshot' },
      writeConcern: { w: 'majority', j: true, wtimeout: 10000 },
      readPreference: 'primary'
    })
    let res
    try { res = await body(session.getDatabase('hw2')) }
    catch (e) { await session.abortTransaction(); throw e }
    await session.commitTransaction()
    return res
  } finally { await session.endSession() }
}

const txSnap = () => {
  const s = db.getSiblingDB('admin').runCommand({ serverStatus: 1, transactions: 1 })
  return {
    host: s.host, pid: s.pid, uptime: s.uptime, localTime: s.localTime,
    totalStarted: s.transactions.totalStarted,
    totalCommitted: s.transactions.totalCommitted,
    totalAborted: s.transactions.totalAborted
  }
}

installValidator(db)

// Исходное состояние каталога (A и B) и заказов:
db.products.find({ _id: { $in: ['A', 'B'] } }).toArray()
db.orders.find().toArray()

// Откат, недостаточно товара B
const rbBefore = txSnap()
try {
  purchase(
    { operationId: 'rollback-001', userId: 2500, items: [{ sku: 'A', size: 'M', qty: 2 }, { sku: 'B', size: 'L', qty: 6 }] },
    e => printjson(e)
  )
} catch (e) {
  print('ERROR CODE:', e.code)
}
const rbAfter = txSnap()
printjson({ before: rbBefore, after: rbAfter })

// Успешная покупка
const okBefore = txSnap()
const savedOrder = purchase(
  { operationId: 'purchase-001', userId: 2500, items: [{ sku: 'A', size: 'M', qty: 2 }, { sku: 'B', size: 'L', qty: 1 }] },
  e => printjson(e)
)
const okAfter = txSnap()

printjson(savedOrder)
printjson({ before: okBefore, after: okAfter })

// Повтор подтверждённой покупки
purchase({
  userId: 2500,
  items: [{ qty: 2, size: 'M', sku: 'A' }, { qty: 1, size: 'L', sku: 'B' }],
  operationId: 'purchase-001'
})

// Конфликт параметров

try {
  purchase({
    operationId: 'purchase-001',
    userId: 2500,
    items: [{ sku: 'A', size: 'M', qty: 3 }, { sku: 'B', size: 'L', qty: 1 }]
  })
} catch (e) {
  print('CONFLICT CODE:', e.code)
}

```

### Ответы на теоритические вопрсы

> различие атомарности и защиты от повторного бизнес-запроса  

Ключевое различие в том, что атомарность прежде всего защищает от частичного выполения запроса.  
Защита от повторного бизнес-запроса нужна, чтобы соблюдалась согласованность с намерениями клиента - операция должна быть проведена 1 раз, если клиент намеревался провести её только 1 раз  

> настройки транзакции и то, как ваш код приводит к commit или abort
Настройки транзакции:  
- readConcern: { level: 'snapshot' }
- writeConcern: { w: 'majority', j: true, wtimeout: 10000 } 
- readPreference: 'primary' - читаем только с праймари, так как в финансовой операции нам важна согласованность и нужны самые актуальные данные

Код приводит к abort выбросом исключения HWError, происходит откат  
Код приводит к commit при успешном завершении функции. Далее обертка transaction сама вызовет commit

## Смена primary  
### Опыт 

Для проведения автоматической проверки использовал  
```
docker compose run --rm lab failover
```

Пошагово вручную проверял следующим образом:  
```
docker compose run --rm lab reset
docker compose run --rm lab shell

load('/work/solution.js')
const c = db.getMongo().getDB('hw2_failover')
hw2rs [primary] hw2> c.adminCommand({ replSetGetStatus: 1 }).members.map(m => ({ host: m.name, role: m.stateStr }))
[
  { host: 'm1:27017', role: 'SECONDARY' },
  { host: 'm2:27017', role: 'SECONDARY' },
  { host: 'm3:27017', role: 'PRIMARY' }
]

writeProbe(c, 'before', 'majority confirmed before stepDown')
{
  acknowledged: true,
  insertedId: null,
  matchedCount: 1,
  modifiedCount: 0,
  upsertedCount: 0
}

try { rs.stepDown(60, 10) } catch (e) { print(e.message) } 
{
  ok: 1,
  '$clusterTime': {
    clusterTime: Timestamp({ t: 1792860410, i: 1 }),
    signature: {
      hash: Binary.createFromBase64('AAAAAAAAAAAAAAAAAAAAAAAAAAA=', 0),
      keyId: Long('0')
    }
  },
  operationTime: Timestamp({ t: 1792860410, i: 1 })
}

c.adminCommand({ replSetGetStatus: 1 }).members.map(m => ({ host: m.name, rrole: m.stateStr }))
[
  { host: 'm1:27017', role: 'PRIMARY' },
  { host: 'm2:27017', role: 'SECONDARY' },
  { host: 'm3:27017', role: 'SECONDARY' }
]

hw2rs [primary] hw2> readProbe(c, 'before')
{ _id: 'before', message: 'majority confirmed before stepDown' }

hw2rs [primary] hw2>  writeProbe(c, 'after', 'majority confirmed after stepDown')
{
  acknowledged: true,
  insertedId: null,
  matchedCount: 1,
  modifiedCount: 0,
  upsertedCount: 0
}

hw2rs [primary] hw2> c.probe.find().toArray()
[
  { _id: 'before', message: 'majority confirmed before stepDown' },
  { _id: 'after', message: 'majority confirmed after stepDown' }
]
```

### Ответы на теоритические вопрсы
> как клиент находит новый primary

Драйвер держит фоновые соединения с известными узлами replica set и периодически отправляет узлам комнды hello, в которых узлы сообщают свои роли.

> почему majority-подтверждение не гарантирует немедленно актуальное чтение с любой secondary

Для достижения majority в mongo достаточно, чтобы на нужном большинстве запись изменения в oplog была устойчиво сохранено. Применение при этом не требуется. 
Поэтому при чтении с любой secondary можем получить ситуацию, что какой-то secondary отстает и ещё ждет изменений, а другой получил, но не успел применить изменение из oplog. И по этой причине нет гарантии, что с majority-подтверждением мы сразу можем читать с любой secondary реплики

## Шардирование

> Какие шарды опросит mongos для истории {userId: 2500}, а какие — для общего отчёта по {status: "paid"}? Обоснуйте оба ответа.

Так как `userId` является ключом шардирования, mongos выполнит адресный запрос только к шарду B, потому что по карте именно шард B владеет диапазоном `[2000, 5000)`.

Статус не является ключом шардирования, документы со статусами paid могут находиться на обоих шардах A и B. mongos выполнит scatter-gather: 
то есть отправит запрос на все шарды, получит результаты и объединит их для клиента. 

> Изменится ли число участников второго запроса после создания индекса {status: 1} внутри шардов? Какую работу этот индекс способен изменить?

Число участников не изменится, так как status всё ещё не является ключом шардирования. Индекс на статус повлияет на локальный план выполнения шарда 
и позволит ускорить выполнение второго запроса на каждом из шардов за счет того, что вместо фулл скана будет использоваться скан по индексу.
