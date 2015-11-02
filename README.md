# ScalaFunctional
[<img src="https://travis-ci.org/EntilZha/ScalaFunctional.svg?branch=master"/>](https://travis-ci.org/EntilZha/ScalaFunctional)
[![Coverage Status](https://coveralls.io/repos/EntilZha/ScalaFunctional/badge.svg?branch=master&service=github)](https://coveralls.io/r/EntilZha/ScalaFunctional?branch=master)
[![ReadTheDocs](https://readthedocs.org/projects/scalafunctional/badge/?version=latest)](https://readthedocs.org/projects/scalafunctional/)
[![Latest Version](https://badge.fury.io/py/scalafunctional.svg)](https://pypi.python.org/pypi/scalafunctional/)
[![Gitter](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/EntilZha/ScalaFunctional?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge)

## Introduction
`ScalaFunctional` is a Python package that makes working with data easy. It takes inspiration from
several sources that include Scala collections, Apache Spark RDDs, Microsoft LINQ and more generally
functional programming. The combination of these ideas makes `ScalaFunctional` a great choice
for declarative transformation and analysis of data.

[Original blog post for ScalaFunctional](http://entilzha.github.io/blog/2015/03/14/functional-programming-collections-python/)

## Installation
`ScalaFunctional` is available on [pypi](https://pypi.python.org/pypi/ScalaFunctional) and can be installed by running:
```bash
# Install from command line
$ pip install scalafunctional
```

Then in python run: `from functional import seq`

## Examples
`ScalaFunctional` is useful for many tasks, and can natively open several common file types. Here
are a few examples of what you can do.

### Simple Example
```python
from functional import seq

seq(1, 2, 3, 4)\
    .map(lambda x: x * 2)\
    .filter(lambda x: x > 4)\
    .reduce(lambda x, y: x + y)
# 14
```

### Filtering a list of account transactions
```python
from functional import seq
from collections import namedtuple

Transaction = namedtuple('Transaction', 'reason amount')
transactions = [
    Transaction('github', 7),
    Transaction('food', 10),
    Transaction('coffee', 5),
    Transaction('digitalocean', 5),
    Transaction('food', 5),
    Transaction('riotgames', 25),
    Transaction('food', 10),
    Transaction('amazon', 200),
    Transaction('paycheck', -1000)
]

# Using the Scala/Spark inspired APIs
food_cost = seq(transactions)\
    .filter(lambda x: x.reason == 'food')\
    .map(lambda x: x.amount).sum()

# Using the LINQ inspired APIs
food_cost = seq(transactions)\
    .where(lambda x: x.reason == 'food')\
    .select(lambda x: x.amount).sum()

# Using ScalaFunctional with fn
from fn import _
food_cost = seq(transactions).filter(_.reason == 'food').map(_.amount).sum()
```

### Word Count and Joins
Tse account transactions example could be done easily in pure python using list comprehensions. To
show some of the things `ScalaFunctional` excels at, take a look at a couple of word count examples.

```python
words = 'I dont want to believe I want to know'.split(' ')
seq(words).map(lambda word: (word, 1)).reduce_by_key(lambda x, y: x + y)
# [('dont', 1), ('I', 2), ('to', 2), ('know', 1), ('want', 2), ('believe', 1)]
```

In the next example we have chat logs formatted in jsonl which contain messages and metadata. A
typical json file will have one valid json on each line of a file. Below are a few lines out of
`examples/chat_logs.jsonl`.

```jsonl
{"message":"hello anyone there?","date":"10/09","user":"bob"}
{"message":"need some help with a program","date":"10/09","user":"bob"}
{"message":"sure thing. What do you need help with?","date":"10/09","user":"dave"}
```

```python
from operator import add
import re
messages = seq.jsonl('examples/chat_lots.jsonl')

# Split words on space and normalize before doing word count
def extract_words(message):
    return re.sub('[^0-9a-z ]+', '', message.lower()).split(' ')


word_counts = messages\
    .map(lambda log: extract_words(log['message']))\
    .flatten().map(lambda word: (word, 1))\
    .reduce_by_key(add).order_by(lambda x: x[1])

```

Next, lets continue that example but introduce a json database of users from `examples/users.json`.
In the previous example we showed how `ScalaFunctional` can do word counts, in the next example lets
show how `ScalaFunctional` can join different data sources.

```python
# First read the json file
users = seq.json('examples/users.json')
#[('sarah',{'date_created':'08/08','news_email':True,'email':'sarah@gmail.com'}),...]

email_domains = users.map(lambda u: u[1]['email'].split('@')[1]).distinct()
# ['yahoo.com', 'python.org', 'gmail.com']

# Join users with their messages
message_tuples = messages.group_by(lambda m: m['user'])
data = users.inner_join(message_tuples)
# [('sarah', 
#    (
#      {'date_created':'08/08','news_email':True,'email':'sarah@gmail.com'},
#      [{'date':'10/10','message':'what is a...','user':'sarah'}...]
#    )
#  ),...]

# From here you can imagine doing more complex analysis
```

### CSV, Aggregate Functions, and Set functions
In `examples/camping_purchases.csv` there are a list of camping purchases. Lets do some cost analysis and
compare it the required camping gear list stored in `examples/gear_list.txt`.

```python
purchases = seq.csv('examples/camping_purchases.csv')
total_cost = purchases.select(lambda row: int(row[2])).sum()
# 1275

most_expensive_item = purchases.max_by(lambda row: int(row[2]))
# ['4', 'sleeping bag', ' 350']

purchased_list = purchases.select(lambda row: row[1])
gear_list = seq.open('examples/gear_list.txt').map(lambda row: row.strip())
missing_gear = gear_list.difference(purchased_list)
# ['water bottle','gas','toilet paper','lighter','spoons','sleeping pad',...]
```

In addition to the aggregate functions show above (`sum` and `max_by`) there are many more.
Similarly, there are several more set like functions in addition to `difference`.

### Writing to files
Just as `ScalaFunctional` can read from `csv`, `json`, `jsonl`, and text files, it can also write
them. For complete API documentation see the collections API table or the official docs.



## Documentation
Summary documentation is below and full documentation is at
[scalafunctional.readthedocs.org](http://scalafunctional.readthedocs.org/en/latest/functional.html).

### Streams, Transformations and Actions
`ScalaFunctional` has three types of functions:

1. Streams: read data for use by the collections API.
2. Transformations: transform data from streams with functions such as `map`, `flat_map`, and `filter`
3. Actions: These cause a series of transformations to evaluate to a concrete value. `to_list`, `reduce`, and `to_dict` are examples of actions.

In the expression `seq(1, 2, 3).map(lambda x: x * 2).reduce(lambda x, y: x + y)`, `seq` is the
stream, `map` is the transformation, and `reduce` is the action.

### Streams API
All of `ScalaFunctional` streams can be accessed through the `seq` object. The primary way to create
a stream is by calling `seq` with an iterable. The `seq` callable is smart and is able to accept
multiple types of parameters as shown in the examples below.

```python
# Passing a list
seq([1, 1, 2, 3]).to_set()
# [1, 2, 3]

# Passing direct arguments
seq(1, 1, 2, 3).map(lambda x: x).to_list()
# [1, 1, 2, 3]

# Passing a single value
seq(1).map(lambda x: -x).to_list()
# [-1]
```

`seq` also provides entry to other streams as attribute functions as shown below.

```python
# number range
seq.range(10)

# text file
seq.open('filepath')

# json file
seq.json('filepath')

# jsonl file
seq.jsonl('filepath')

# csv file
seq.csv('filepath')
```

For more information on the parameters that these functions can take, reference the
[streams documentation](http://scalafunctional.readthedocs.org/en/latest/functional.html#module-functional.streams)

### Transformations and Actions APIs
Below is the complete list of functions which can be called on a stream object from `seq`. For
complete documentation reference 
[transformation and actions API](http://scalafunctional.readthedocs.org/en/latest/functional.html#module-functional.pipeline).

Function | Description | Type
 ------- | -----------  | ----
`map(func)/select(func)` | Maps `func` onto elements of sequence | transformation
`filter(func)/where(func)` | Filters elements of sequence to only those where `func(element)` is `True` | transformation
`filter_not(func)` | Filters elements of sequence to only those where `func(element)` is `False` | transformation
`flatten()` | Flattens sequence of lists to a single sequence | transformation
`flat_map(func)` | `func` must return an iterable. Maps `func` to each element, then merges the result to one flat sequence | transformation
`group_by(func)` | Groups sequence into `(key, value)` pairs where `key=func(element)` and `value` is from the original sequence | transformation
`group_by_key()` | Groups sequence of `(key, value)` pairs by `key` | transformation
`reduce_by_key(func)` | Reduces list of `(key, value)` pairs using `func` | transformation
`union(other)` | Union of unique elements in sequence and `other` | transformation
`intersection(other)` | Intersection of unique elements in sequence and `other` | transformation
`difference(other)` | New sequence with unique elements present in sequence but not in `other` | transformation
`symmetric_difference(other)` | New sequence with unique elements present in sequnce or `other`, but not both | transformation
`distinct()` | Returns distinct elements of sequence. Elements must be hashable | transformation
`distinct_by(func)` | Returns distinct elements of sequence using `func` as a key | transformation
`drop(n)` | Drop the first `n` elements of the sequence | transformation
`drop_right(n)` | Drop the last `n` elements of the sequence | transformation
`drop_while(func)` | Drop elements while `func` evaluates to `True`, then returns the rest | transformation
`take(n)` | Returns sequence of first `n` elements | transformation
`take_while(func)` | Take elements while `func` evaluates to `True`, then drops the rest | transformation
`init()` | Returns sequence without the last element | transformation
`tail()` | Returns sequence without the first element | transformation
`inits()` | Returns consecutive inits of sequence | transformation
`tails()` | Returns consecutive tails of sequence | transformation
`zip(other)` | Zips the sequence with `other` | transformation
`zip_with_index()` | Zips the sequence with the index starting at zero on the left side | transformation
`enumerate(start=0)` | Zips the sequence with the index starting at `start` on the left side | transformation
`inner_join(other)` | Returns inner join of sequence with other. Must be a sequence of `(key, value)` pairs | transformation
`outer_join(other)` | Returns outer join of sequence with other. Must be a sequence of `(key, value)` pairs | transformation
`left_join(other)` | Returns left join of sequence with other. Must be a sequence of `(key, value)` pairs | transformation
`right_join(other)` | Returns right join of sequence with other. Must be a sequence of `(key, value)` pairs | transformation
`join(other, join_type='inner')` | Returns join of sequence with other as specified by `join_type`. Must be a sequence of `(key, value)` pairs | transformation
`partition(func)` | Partitions the sequence into elements which satisfy `func(element)` and those that don't | transformation
`grouped(size)` | Partitions the elements into groups of size `size` | transformation
`sorted(key=None, reverse=False)/order_by(func)` | Returns elements sorted according to python `sorted` | transformation
`reverse()` | Returns the reversed sequence | transformation
`slice(start, until)` | Sequence starting at `start` and including elements up to `until` | transformation
`head()` / `first()` | Returns first element in sequence | action
`head_option()` | Returns first element in sequence or `None` if its empty | action
`last()` | Returns last element in sequence | action
`last_option()` | Returns last element in sequence or `None` if its empty | action
`len()` / `size()` | Returns length of sequence | action
`count(func)` | Returns count of elements in sequence where `func(element)` is True | action
`empty()` | Returns `True` if the sequence has zero length | action
`non_empty()` | Returns `True` if sequence has non-zero length | action
`all()` | Returns `True` if all elements in sequence are truthy | action
`exists(func)` | Returns `True` if `func(element)` for any element in the sequence is `True` | action
`for_all(func)` | Returns `True` if `func(element)` is `True` for all elements in the sequence | action
`find(func)` | Returns the element that first evaluates `func(element)` to `True` | action
`any()` | Returns `True` if any element in sequence is truthy | action
`max()` | Returns maximal element in sequence | action
`min()` | Returns minimal element in sequence | action
`max_by(func)` | Returns element with maximal value `func(element)` | action
`min_by(func)` | Returns element with minimal value `func(element)` | action
`sum()` | Returns the sum of elements | action
`product()` | Returns the product of elements | action
`average()` | Returns the average of elements | action
`aggregate(func)/aggregate(seed, func)/aggregate(seed, func, result_map)` | Aggregate using `func` starting with `seed` or first element of list then apply `result_map` to the result | action
`fold_left(zero_value, func)` | Reduces element from left to right using `func` and initial value `zero_value` | action
`fold_right(zero_value, func)` | Reduces element from right to left using `func` and initial value `zero_value` | action
`make_string(separator)` | Returns string with `separator` between each `str(element)` | action
`dict(default=None)` / `to_dict(default=None)` | Converts a sequence of `(Key, Value)` pairs to a `dictionary`. If `default` is not None, it must be a value or zero argument callable which will be used to create a `collections.defaultdict` | action
`list()` / `to_list()` | Converts sequence to a list | action
`set() / to_set()` | Converts sequence to a set | action
`to_file(path)` | Saves the sequence to a file at path with each element on a newline | action
`to_csv(path)` | Saves the sequence to a csv file at path with each element representing a row | action
`to_jsonl(path)` | Saves the sequence to a jsonl file with each element being transformed to json and printed to a new line | action
`to_json(path)` | Saves the sequence to a json file. The contents depend on if the json root is an array or dictionary | action
`cache()` | Forces evaluation of sequence immediately and caches the result | action
`for_each(func)` | Executes `func` on each element of the sequence | action


## Road Map
* Parallel execution engine for faster computation `0.5.0`
* SQL based query planner and interpreter (TBD on if/when/how this would be done)
* When is this ready for `1.0`?
* Perhaps think of a better name that better suits this package than `ScalaFunctional`

## Contributing and Bug Fixes
Any contributions or bug reports are welcome. Thus far, there is a 100% acceptance rate for pull
requests and contributors have offered valuable feedback and critique on code. It is great to hear
from users of the package, especially what it is used for, what works well, and what could be
improved.

## Contact/Email list/Chat
[Google Groups mailing list](https://groups.google.com/forum/#!forum/scalafunctional)

[Gitter for chat](https://gitter.im/EntilZha/ScalaFunctional)

## Supported Python Versions
`ScalaFunctional` supports and is tested against Python 2.7, 3.3, 3.4, 3.5, PyPy, and PyPy3

## Changelog
[Changelog](https://github.com/EntilZha/ScalaFunctional/CHANGELOG.md)

