# Node Parser
Cycle ORM provides convinient* way to convert flat structures into data trees. Parser can work using one large query or multiple queries
using identical approach. Parses work with numeric arrays.

> This section is intended for advanced scenarios, make sure you can't achieve given flexibility using default instuments before jumping to this approach.

## Upack simple query
We can start with simple example which converts query result into associated array (example is using Database instance):

```php
$query = $db->select('id', 'balance')->from('users');
```

The simple node parser will look like:

```php
$root = new RootNode(
    ['id', 'balance'], // property names
    'id'               // primary key
);

foreach ($query->run()->fetchAll(StatementInterface::FETCH_NUM) as $row) {
    // start from 1st (0) column
    $root->parseRow(0, $row);
}

print_r($root->getResult());
```

Given code does not do much, but we can use it now to perform more complex transformation. For example we can join some extenrnal table to our query:

```php
$query = $db
    ->select('u.id', 'u.balance', 'o.id', 'o.user_id', 'o.total')
    ->from('users as u')
    ->leftJoin('orders as o')->on('o.user_id', 'u.id');
```

The query will return results in a form: [user.id, user.balance, order.id, order.user_id, order.total]. Lets upack it into structure like:

```php
[
    [
        'id'      => 1,
        'balance' => 10,
        'orders'  => [
            [
                'id'      => 1,
                'user_id' => 1,
                'total'   => 100
            ],
            // ...
        ]
    ]
]
```

Since both tables are merged in one query we have to create and join sub node (array):

```php
$root = new RootNode(
    ['id', 'balance'],  // property names
    'id'                // primary key
);

$root->joinNode('orders', new ArrayNode(
    ['id', 'user_id', 'total'], // property names
    'id',                       // primary key
    'user_id',                  // inner key
    'id'                        // outer key (user.id)
));

foreach ($query->run()->fetchAll(StatementInterface::FETCH_NUM) as $row) {
    // start from 1st (0) column
    $root->parseRow(0, $row);
}
```

> Check SingularNode for one to one associations.

## External Queries
In some cases (for example for one-to-many) associations it might be useful to execute relation query using external SQL SELECT and `WHERE IN` statement. This can be achieved by linking nodes together to aggregate query context:

```php
$query = $db->select('u.id', 'u.balance')->from('users as u');

$root = new RootNode(
    ['id', 'balance'],  // property names
    'id'                // primary key
);

$orders = new ArrayNode(
    ['id', 'user_id', 'total'], // property names
    'id',                       // primary key
    'user_id',                  // inner key
    'id'                        // outer key (user.id)
);

// notice the change
$root->linkNode('orders', $orders);

foreach ($query->run()->fetchAll(StatementInterface::FETCH_NUM) as $row) {
    // start from 1st (0) column
    $root->parseRow(0, $row);
}
```

Now, `orders` array in our structure would not be populated, but we can request list of collected ids from the root loader:

```php
// only populated after parsing all the rows by the root node
print_r($orders->getReferences());
```

We can use this references (user.id) to create orders query:

```php
$query = $db
    ->select('o.id', 'o.user_id', 'o.total')
    ->from('orders as o')
    ->where('o.user_id', 'in', new Parameter($orders->getReferences()));

foreach ($query->run()->fetchAll(StatementInterface::FETCH_NUM) as $row) {
    // start from 1st (0) column
    $orders->parseRow(0, $row);
}
```

Order node will mount it's parsed entities to root node automatically, giving us identical result as in case with `LEFT JOIN`:

```php
print_r($root->getResult();
```