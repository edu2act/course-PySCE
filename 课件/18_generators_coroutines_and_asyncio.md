# 生成器、协程和asyncio

- [生成器](#org86d5de7)
  - [生成器表达式](#org203f341)
  - [定义生成器](#org7dd10b4)
  - [关闭生成器对象](#org1841608)
- [协程与 `yield` 表达式](#org8e56303)
- [生成器示例](#org0c0afe4)
  - [示例2](#orgb55797c)
  - [示例3](#org3154c20)
  - [示例4](#org0baf12d)
- [协程示例](#org43267b4)
  - [示例2](#org1368bbc)
- [`yield from`](#org2a6b5af)
  - [示例](#org5a31afb)
- [使用协程进行并发编程](#org6973cf0)
  - [运行协程](#orgcb4ffb3)
- [回调](#org88aa05a)
- [多个协程](#org0d9af90)
- [`run_until_complete` / `run_forever`](#orgb13cf27)
- [关闭 loop](#orgda833fd)
- [coroutine 和 future](#org4c182c8)
- [Task 任务退出](#org6cc64a3)
- [Timer](#orgfa4423f)
- [`async for` / `async with`](#org3f10800)
- [示例](#orgca78bf4)
  - [示例2](#org842917a)
  - [示例3](#org1a049ea)
  - [示例4](#org8595411)



<a id="org86d5de7"></a>

# 生成器

生成器是一个函数，它生成一个值的序列，可以在迭代中使用。


<a id="org203f341"></a>

## 生成器表达式

语法：<code>(expr for item_var in iterable if cond_expr)</code>

```python
num = (x for x in range(10000) if x * x == (x - 3) * (x + 4))
print(repr(type(num)))
print(next(num))
```

    <class 'generator'>
    12


<a id="org7dd10b4"></a>

## 定义生成器

使用<code>yield</code>关键字来定义生成器对象：

```ipython
def countdown(n):
    print('Count down from %d' % n)
    while n > 0:
        yield n
        n -= 1
    return

c = countdown(10)
print(type(c))
print(c.__next__())
print(next(c))
```

    <class 'generator'>
    Count down from 10
    10
    9

调用生成器对象的<code>__next__()</code>方法或者使用内建函数<code>next()</code>时，生成器函数将开始执行语句，直到遇到<code>yield</code>返回，<code>yield</code>语句在函数执行停止的地方生成一个结果，直到下次调用<code>__next__()</code>（或<code>next()</code>），之后继续执行<code>yield</code>之后的语句。

```ipython
for n in countdown(10):
    print(n)

a = sum(countdown(10))
print(a)
```

    Count down from 10
    10
    9
    8
    7
    6
    5
    4
    3
    2
    1
    Count down from 10
    55

生成器函数完成的标志是返回或引发<code>StopIteration</code>异常。


<a id="org1841608"></a>

## 关闭生成器对象

```ipython
for n in countdown(10):
    if n == 2:
        break
    print(n)
```

```ipython
c = countdown(10)
c.__next__()
c.close()     # will close the generator
c.__next__()  # StopIteration
```

对于不再使用或删除生成器时，可以调用<code>close()</code>方法。

另外，在生成器函数内部，当生成器已通过<code>close()</code>关闭时，在<code>yield</code>语句上出现<code>GeneratorExit</code>异常，可以选择捕捉这个异常，以便执行清理工作。

```ipython
def countdown(n):
    print('Counting down from %d' % n)
    try:
        while n > 0:
            yield n
            n -= 1
    except GeneratorExit:
        print('Only made it to %d' % n)

c = countdown(5)
for num in c:
    print(num)
    if num == 3:
        c.close()
```

    Counting down from 5
    5
    4
    3
    Only made it to 3

一般情况，对于生成器函数，虽然可以捕捉<code>GeneratorExit</code>异常，但是通过<code>yield</code>语句处理异常并返回另一个输出值时不合法的。另外，如果程序当前正在对生成器进行迭代，不应通过另一个执行线程或异步调用该生成器的<code>close()</code>方法。


<a id="org8e56303"></a>

# 协程与 `yield` 表达式

```ipython
def receiver():
    print('Ready to receive')
    while 1:
        n = yield
        print('Got %s' % n)

r = receiver()
r.__next__()  # 向前执行到第一条 yield 语句
r.send(1)
r.send(2)
r.send('hello')
```

    Ready to receive
    Got 1
    Got 2
    Got hello

以这种方式使用<code>yield</code>语句的函数称为 **协程** 。向函数发送值时函数将执行，传递给<code>send()</code>的值由协程中的<code>yield</code>表达式返回。接收到值时，协程就会执行语句，直至遇到下一条<code>yield</code>语句。

不要忽略首先调用<code>__next__()</code>。<code>__next__()</code>和<code>send()</code>在一定意义上作用是相似的，区别是<code>send()</code>可以传递<code>yield</code>表达式的值进去，而<code>__next__()</code>不能传递特定的值，只能传递<code>None</code>进去。因此，<code>__next__()</code>和<code>send(None)</code>作用是一样的。

```ipython
def coroutine(func):
    def wrapper(*args, **kwargs):
        r = func(*args, **kwargs)
        r.__next__()
        return r
    return wrapper

@coroutine
def receiver():
    print('Ready to receive')
    while 1:
        n = yield
        print('Got %s' % n)

r = receiver()
r.send('hello, world!')  # 无需初始调用 __next__()
```

    Ready to receive
    Got hello, world!

协程一般会不断地执行下去，除非被显式关闭或者自己退出。

```ipython
r.close()
r.send(4)  # StopIteration
```

跟前面生成器一样，<code>close()</code>操作将在协程内部引发<code>GeneratorExit</code>异常。

```ipython
@coroutine
def receiver():
    print('Ready to receive')
    try:
        while 1:
            n = yield
            print('Got %s' % n)
    except GeneratorExit:
        print('Receive done')


r = receiver()
r.send('hello, world')
r.close()
```

    Ready to receive
    Got hello, world
    Receive done

可以使用<code>throw(type[, value[, traceback]])</code>方法在协程内部引发异常。

```ipython
r = receiver()
r.send('hello, world')
r.throw(RuntimeError, "You're hosed!")  # RuntimeError: You're hosed!
```

协程可以选择捕捉异常并以正确的方式处理它们。使用<code>throw()</code>方法作为给协程的异步信号并不安全——永远都不应该通过单独的执行线程或以异步方式调用这个方法。

如果<code>yield</code>表达式中提供了值，协程可以使用<code>yield</code>语句接收和发出返回值。

```ipython
def line_splitter(delimiter=None):
    print('Ready to split')
    result = None
    while 1:
        line = yield result
        print(line)
        result = line.split(delimiter)
        print('result: ', result)


s = line_splitter(',')
print(s.__next__())  # 返回 result 的初始值 None
print('begin splitting')
print('send: ', s.send('A,B,C'))
print('send: ', s.send('100, 200, 300'))
```

    Ready to split
    None
    begin splitting
    A,B,C
    result:  ['A', 'B', 'C']
    send:  ['A', 'B', 'C']
    100, 200, 300
    result:  ['100', ' 200', ' 300']
    send:  ['100', ' 200', ' 300']


<a id="org0c0afe4"></a>

# 生成器示例

```ipython
import os
import fnmatch
import gzip
import bz2


def find_files(topdir, pattern, recursive=False):
    if recursive:
        for path, dirname, filelist in os.walk(topdir):
            for name in filelist:
                if fnmatch.fnmatch(name, pattern):
                    yield os.path.join(path, name)
    else:
        for name in os.listdir(topdir):
            file_path = os.path.join(topdir, name)
            if os.path.isfile(file_path):
                if fnmatch.fnmatch(file_path, pattern):
                    yield file_path


def opener(filenames):
    for name in filenames:
        if name.endswith('.gz'):
            f = gzip.open(name)
        elif name.endswith('.bz2'):
            f = bz2.BZ2File(name)
        else:
            f = open(name)
        yield name, f


def cat(filelist):
    for name, f in filelist:
        for line in f:
            yield name, line


def grep(pattern, lines):
    for name, line in lines:
        if pattern in line:
            yield name, line
```

```ipython
import os

home = os.path.expanduser('~')
bash_files = find_files(home, '*bash*')
files = opener(bash_files)
lines = cat(files)
pip_lines = grep('brew', lines)
for i, line in enumerate(pip_lines):
    if 15 < i < 21:
        print(*line, sep=': ')
```

    /Users/fishtai0/.bash_history: brew info git

    /Users/fishtai0/.bash_history: brew list

    /Users/fishtai0/.bash_history: brew install libxml2

    /Users/fishtai0/.bash_history: brew update

    /Users/fishtai0/.bash_history: brew outdated


<a id="orgb55797c"></a>

## 示例2

```ipython
def countdown(n):
    while n > 0:
        print('T-minus', n)
        yield
        n -= 1
        print('Blast off!')

def countup(n):
    x = 0
    while x < n:
        print('Counting up', x)
        yield
        x += 1


from collections import deque

class TaskScheduler:
    def __init__(self):
        self._task_queue = deque()

    def new_task(self, task):
        self._task_queue.append(task)

    def run(self):
        while self._task_queue:
            task = self._task_queue.popleft()
            try:
                next(task)  # activate generator
                self._task_queue.append(task)
            except StopIteration:
                pass


sched = TaskScheduler()
sched.new_task(countdown(10))
sched.new_task(countdown(5))
sched.new_task(countup(15))
sched.run()
```

    T-minus 10
    T-minus 5
    Counting up 0
    Blast off!
    T-minus 9
    Blast off!
    T-minus 4
    Counting up 1
    Blast off!
    T-minus 8
    Blast off!
    T-minus 3
    Counting up 2
    Blast off!
    T-minus 7
    Blast off!
    T-minus 2
    Counting up 3
    Blast off!
    T-minus 6
    Blast off!
    T-minus 1
    Counting up 4
    Blast off!
    T-minus 5
    Blast off!
    Counting up 5
    Blast off!
    T-minus 4
    Counting up 6
    Blast off!
    T-minus 3
    Counting up 7
    Blast off!
    T-minus 2
    Counting up 8
    Blast off!
    T-minus 1
    Counting up 9
    Blast off!
    Counting up 10
    Counting up 11
    Counting up 12
    Counting up 13
    Counting up 14


<a id="org3154c20"></a>

## 示例3

```ipython
def lucas():
    yield 2
    a = 2
    b = 1
    while 1:
        yield b
        a, b = b, a + b


from itertools import islice
print(list(islice(lucas(), 10)))


def search(iterable, predicate):
    """Returns the first item satisfying a predicate
    """
    for item in iterable:
        if predicate(item):
            return item
    raise ValueError('Not found')

print(search(lucas(), lambda x: len(str(x)) >= 6))

def async_search(iterable, predicate):
    """Final result 'returned' in StopIteration payload
    """
    for item in iterable:
        if predicate(item):
            return item
        yield

    raise ValueError('Not found')


g = async_search(lucas(), lambda x: x > 10)
print(g)
print(id(g))
print(next(g))
```

    [2, 1, 3, 4, 7, 11, 18, 29, 47, 76]
    103682
    <generator object async_search at 0x10625a308>
    4398097160
    None

```ipython
class Task:

    next_id = 0

    def __init__(self, routine):
        self.id = Task.next_id
        Task.next_id += 1
        self.routine = routine


from collections import deque

class Scheduler:

    def __init__(self):
        self.runnable_tasks = deque()
        self.completed_task_results = {}
        self.failed_task_errors = {}

    def add(self, routine):
        task = Task(routine)
        self.runnable_tasks.append(task)
        return task.id

    def run_to_completion(self):
        while len(self.runnable_tasks) != 0:
            task = self.runnable_tasks.popleft()
            print('Running task {} ...'.format(task.id), end='')
            try:
                yielded = next(task.routine)
            except StopIteration as stopped:
                print('completed with result: {!r}'.format(stopped.value))
                self.completed_task_results[task.id] = stopped.value
            except Exception as e:
                print('failed with exception: {}'.format(e))
                self.failed_task_errors[task.id] = e
            else:
                assert yielded is None
                print('now yielded')
                self.runnable_tasks.append(task)

scheduler = Scheduler()
scheduler.add(async_search(lucas(), lambda x: len(str(x)) >= 6))
scheduler.run_to_completion()
print(scheduler.completed_task_results.pop(0))
scheduler.add(async_search(lucas(), lambda x: len(str(x)) >= 7))
scheduler.add(async_search(lucas(), lambda x: len(str(x)) >= 9))
scheduler.run_to_completion()
print(scheduler.completed_task_results.pop(1))
print(scheduler.completed_task_results.pop(2))
```

    Running task 0 ...now yielded
    Running task 0 ...now yielded
    Running task 0 ...now yielded
    Running task 0 ...now yielded
    Running task 0 ...now yielded
    Running task 0 ...now yielded
    Running task 0 ...now yielded
    Running task 0 ...now yielded
    Running task 0 ...now yielded
    Running task 0 ...now yielded
    Running task 0 ...now yielded
    Running task 0 ...now yielded
    Running task 0 ...now yielded
    Running task 0 ...now yielded
    Running task 0 ...now yielded
    Running task 0 ...now yielded
    Running task 0 ...now yielded
    Running task 0 ...now yielded
    Running task 0 ...now yielded
    Running task 0 ...now yielded
    Running task 0 ...now yielded
    Running task 0 ...now yielded
    Running task 0 ...now yielded
    Running task 0 ...now yielded
    Running task 0 ...completed with result: 103682
    103682
    Running task 1 ...now yielded
    Running task 2 ...now yielded
    Running task 1 ...now yielded
    Running task 2 ...now yielded
    Running task 1 ...now yielded
    Running task 2 ...now yielded
    Running task 1 ...now yielded
    Running task 2 ...now yielded
    Running task 1 ...now yielded
    Running task 2 ...now yielded
    Running task 1 ...now yielded
    Running task 2 ...now yielded
    Running task 1 ...now yielded
    Running task 2 ...now yielded
    Running task 1 ...now yielded
    Running task 2 ...now yielded
    Running task 1 ...now yielded
    Running task 2 ...now yielded
    Running task 1 ...now yielded
    Running task 2 ...now yielded
    Running task 1 ...now yielded
    Running task 2 ...now yielded
    Running task 1 ...now yielded
    Running task 2 ...now yielded
    Running task 1 ...now yielded
    Running task 2 ...now yielded
    Running task 1 ...now yielded
    Running task 2 ...now yielded
    Running task 1 ...now yielded
    Running task 2 ...now yielded
    Running task 1 ...now yielded
    Running task 2 ...now yielded
    Running task 1 ...now yielded
    Running task 2 ...now yielded
    Running task 1 ...now yielded
    Running task 2 ...now yielded
    Running task 1 ...now yielded
    Running task 2 ...now yielded
    Running task 1 ...now yielded
    Running task 2 ...now yielded
    Running task 1 ...now yielded
    Running task 2 ...now yielded
    Running task 1 ...now yielded
    Running task 2 ...now yielded
    Running task 1 ...now yielded
    Running task 2 ...now yielded
    Running task 1 ...now yielded
    Running task 2 ...now yielded
    Running task 1 ...now yielded
    Running task 2 ...now yielded
    Running task 1 ...now yielded
    Running task 2 ...now yielded
    Running task 1 ...now yielded
    Running task 2 ...now yielded
    Running task 1 ...now yielded
    Running task 2 ...now yielded
    Running task 1 ...now yielded
    Running task 2 ...now yielded
    Running task 1 ...completed with result: 1149851
    Running task 2 ...now yielded
    Running task 2 ...now yielded
    Running task 2 ...now yielded
    Running task 2 ...now yielded
    Running task 2 ...now yielded
    Running task 2 ...now yielded
    Running task 2 ...now yielded
    Running task 2 ...now yielded
    Running task 2 ...now yielded
    Running task 2 ...now yielded
    Running task 2 ...completed with result: 141422324
    1149851


<a id="org0baf12d"></a>

## 示例4

```ipython
def async_print_matches(iterable, predicate):
    for item in iterable:
        if predicate(item):
            print('Found:', item, end=', ')
        yield
```


<a id="org43267b4"></a>

# 协程示例

协程可用于编写数据流处理程序。

```ipython
import os
import fnmatch
import gzip
import bz2


@coroutine
def find_files(target, recursive=False):
    while 1:
        topdir, pattern = yield
        if recursive:
            for path, dirname, filelist in os.walk(topdir):
                for name in filelist:
                    if fnmatch.fnmatch(name, pattern):
                        target.send(os.path.join(path, name))
        else:
            for name in os.listdir(topdir):
                file_path = os.path.join(topdir, name)
                if os.path.isfile(file_path):
                    if fnmatch.fnmatch(file_path, pattern):
                        target.send(file_path)


@coroutine
def opener(target):
    while 1:
        name = yield
        if name.endswith('.gz'):
            f = gzip.open(name)
        elif name.endswith('.bz2'):
            f = bz2.BZ2File(name)
        else:
            f = open(name)
        target.send((name, f))


@coroutine
def cat(target):
    while 1:
        name, f = yield
        for line in f:
            target.send((name, line))


@coroutine
def grep(pattern, target):
    while 1:
        name, line = yield
        if pattern in line:
            target.send((name, line))

count = 0
@coroutine
def printer():
    global count
    while 1:
        name, line = yield
        if 15 < count < 21:
            print(name, line, sep=': ')
        count += 1
```

```ipython
finder = find_files(opener(cat(grep('brew', printer()))))

home = os.path.expanduser('~')
finder.send((home, '*bash*'))
```

    /Users/fishtai0/.bash_history: brew info git

    /Users/fishtai0/.bash_history: brew list

    /Users/fishtai0/.bash_history: brew install libxml2

    /Users/fishtai0/.bash_history: brew update

    /Users/fishtai0/.bash_history: brew outdated


<a id="org1368bbc"></a>

## 示例2

```ipython
from collections import namedtuple

Result = namedtuple('Result', 'count average')

def averager():
    total = 0.0
    count = 0
    average = None
    while 1:
        term = yield
        if term is None:
            break
        total += term
        count += 1
        average = total / count
    return Result(count, average)

coro_avg = averager()
next(coro_avg)  # 激活协程
coro_avg.send(10)
coro_avg.send(30)
coro_avg.send(6.5)
coro_avg.send(None)  # StopIteration: Result(count=3, average=15.5)
```

<code>return</code>表达式的值会偷偷传给调用方，赋值给<code>StopIteration</code>异常的一个属性。

获取协程返回的值：

```ipython
coro_avg = averager()
next(coro_avg)
coro_avg.send(10)
coro_avg.send(30)
coro_avg.send(6.5)
try:
    coro_avg.send(None)
except StopIteration as exc:
    result = exc.value

print(result)
```

    Result(count=3, average=15.5)


<a id="org2a6b5af"></a>

# `yield from`

在生成器<code>gen</code>中使用<code>yield from subgen()</code>时，<code>subgen</code>会获得控制权，把产出的值传给<code>gen</code>的调用方，即调用方可直接控制<code>subgen</code>。与此同时，<code>gen</code>会阻塞，等待<code>subgen</code>终止。

```ipython
def chain(*iterables):
    for it in iterables:
        for i in it:
            yield i

def chain2(*iterables):
    for it in iterables:
        yield from it

s = 'ABC'
t = tuple(range(3))
print(list(chain(s, t)))
print(list(chain2(s, t)))
```

    ['A', 'B', 'C', 0, 1, 2]
    ['A', 'B', 'C', 0, 1, 2]

<code>yield from</code>可用于简化<code>for</code>循环中的<code>yield</code>表达式。<code>yield from x</code>表达式对<code>x</code>对象所做的第一件事是，调用<code>iter(x)</code>，从中获取迭代器。<code>x</code>可以是任何可迭代的对象。

将一个多层嵌套的序列展开称一个单层列表：

```ipython
from collections import Iterable


def flatten(items, ignore_types=(str, bytes)):
    for x in items:
        if isinstance(x, Iterable) and not isinstance(x, ignore_types):
            yield from flatten(x)  # for i in flatten(x): yield i
        else:
            yield x


items = [1, 2, [3, 4, [5, 6], 7], 8, 'abc', 'defg', ['hij', 'klmn']]
for x in flatten(items):
    print(x)
```

    1
    2
    3
    4
    5
    6
    7
    8
    abc
    defg
    hij
    klmn

上面示例中，<code>yield from</code>会返回所有子生成器的值。

```ipython
from collections import namedtuple

Result = namedtuple('Result', 'count average')

def averager():
    total = 0.0
    count = 0
    average = None
    while 1:
        term = yield
        if term is None:
            break
        total += term
        count += 1
        average = total / count
    return Result(count, average)


# 委派生成器
def grouper(results, key):
    while 1:
        results[key] = yield from averager()


# 客户端代码，调用方
def main(data):
    results = {}
    for key, values in data.items():
        group = grouper(results, key)
        next(group)
        for value in values:
            group.send(value)
        group.send(None)

    # print(results)
    report(results)

def report(results):
    for key, result in sorted(results.items()):
        group, unit = key.split(';')
        print('{:2} {:5} averaging {:.2f}{}'.format(
            result.count, group, result.average, unit))

data = {
    'girls;kg':
    [40.9, 38.5, 44.3, 42.2, 45.2, 41.7, 44.5, 38.0, 40.6, 44.5],
    'girls;m':
    [1.6, 1.51, 1.4, 1.3, 1.41, 1.39, 1.33, 1.46, 1.45, 1.43],
    'boys;kg':
    [39.0, 40.8, 43.2, 40.8, 43.1, 38.6, 41.4, 40.6, 36.3],
    'boys;m':
    [1.38, 1.5, 1.32, 1.25, 1.37, 1.48, 1.25, 1.49, 1.46],
}

main(data)
```

     9 boys  averaging 40.42kg
     9 boys  averaging 1.39m
    10 girls averaging 42.04kg
    10 girls averaging 1.43m

因为委派生成器相当于管道，所以可以把任意数量个委派生成器链接在一起：一个委派生成器使用<code>yield from</code>调用一个子生成器，而子生成器本身也是委派生成器，使用<code>yield from=调用另一个子生成器，以此类推。最终，这个链条要以一个只使用 =yield</code>表达式的简单生成器（或者任何可迭代的对象）结束。

任何<code>yield from</code>链条都必须由客户驱动，在最外层委派生成器上调用<code>next(...)</code>函数或者<code>.send(...)</code>方法。


<a id="org5a31afb"></a>

## 示例

```ipython
import time

from math import sqrt


def lucas():
    yield 2
    a = 2
    b = 1
    while 1:
        yield b
        a, b = b, a + b


def async_is_prime(x):
    if x < 2:
        return False
    for i in range(2, int(sqrt(x)) + 1):
        if x % i == 0:
            return False
        yield from async_sleep(0)
    return True


def async_search(iterable, async_predicate):
    for item in iterable:
        if (yield from async_predicate(item)):
            return item
    raise ValueError('Not found')


def async_print_matches(iterable, async_predicate):
    for item in iterable:
        matches = yield from async_predicate(item)
        if matches:
            print('Found:', item)


def async_repetitive_message(message,
                             interval_seconds):
    while 1:
        print(message)
        yield from async_sleep(interval_seconds)


def async_sleep(interval_seconds):
    start = time.time()
    expiry = start + interval_seconds
    while 1:
        yield
        now = time.time()
        if now >= expiry:
            break
```


<a id="org6973cf0"></a>

# 使用协程进行并发编程

对 Python 来说，并发（concurrency）不仅可以通过多线程（threading）和多进程（multiprocessing）来实现，还可以使用 **asyncio** 来编写异步程序。

Python 3.4 标准库引入了<code>asyncio</code>，使得可以利用协程编写单线程并发代码，通过I/O多路复用访问套接字和其他资源。

**注意：<code>asyncio</code>并不能带来真正的并行（parallelism）。**

可以将协程任务交给 asyncio 执行。一个协程可以放弃执行，把机会让给其它协程（即<code>yield from</code>或<code>await</code>）。

Python 3.5 之前协程的写法：

```python
# old_coroutine.py
import asyncio

@asyncio.coroutine
def slow_operation(n):
    yield from asyncio.sleep(1)
    print('Slow operation {} complete'.format(n))


@asyncio.coroutine
def main():
    yield from asyncio.wait([
        slow_operation(1),
        slow_operation(2),
        slow_operation(3),
    ])


loop = asyncio.get_event_loop()
loop.run_until_complete(main())
```

    Slow operation 2 complete
    Slow operation 1 complete
    Slow operation 3 complete

调用<code>get_event_loop</code>将返回默认的事件循环，用于负责所有协程的调度。在大量协程并发执行的过程中，除了在协程中主动使用<code>await</code>，当本地协程发生I/O等待时，调用<code>asyncio.sleep</code>，程序的控制权也会在不同的协程间切换，从而在GIL的限制下实现最大程度的并发执行，不会由于等待I/O等原因导致程序阻塞，达到较高的性能。

Python 3.5 添加了<code>async</code>和<code>await</code>这两个关键字，使得协程成为新的语法。

使用<code>async</code>/<code>await</code>关键词之后:

```python
# new_coroutine.py
import asyncio

async def slow_operation(n):
    await asyncio.sleep(1)
    print('Slow operation {} complete'.format(n))


async def main():
    await asyncio.wait([
        slow_operation(1),
        slow_operation(2),
        slow_operation(3),
    ])


loop = asyncio.get_event_loop()
loop.run_until_complete(main())
```

<code>async</code>简化了<code>asyncio.coroutine</code>，<code>await</code>简化了<code>yield from</code>。

上面的例子中，<code>async</code>用于声明一个协程：

```ipython
import asyncio


async def coro(wt):
    print('Waiting ' + str(wt))
    await asyncio.sleep(wt)

print(asyncio.iscoroutinefunction(coro))
```

    True

<code>coro</code>便是一个协程，准确来说，其是一个协程函数，可以通过<code>asyncio.iscoroutinefunction</code>来验证。

协程可以做如下事情：

-   等待一个<code>future</code>结束
-   等待另一个协程（产生一个结果，或引发一个异常）
-   产生一个结果给正在等它的协程
-   引发一个异常给正在等它的协程

<code>await</code>表示等待另一个协程执行完返回，获取协程执行结果，它必须在协程内才能使用。

<code>asyncio.sleep</code>也是一个协程，所以<code>await asyncio.sleep(wt)</code>就是等待另一个协程。


<a id="orgcb4ffb3"></a>

## 运行协程

调用协程函数，协程并不会开始运行，只是返回一个协程对象，可以通过<code>asyncio.iscoroutine</code>来验证：

```ipython
print(asyncio.iscoroutine(coro(3)))
```

要让协程对象运行的话，有两种方式：

1.  在另一个已经运行的协程中用<code>await</code>等待它
2.  通过<code>ensure_future</code>函数计划它的执行

简单来说，只有 **loop** 运行了，协程才可能运行。

先拿到当前线程缺省的 **loop** ，然后把协程对象交给<code>loop.run_until_complete</code>，协程对象随后会在 **loop** 里得到运行。

```ipython
loop = asyncio.get_event_loop()
loop.run_until_complete(coro(3))
```

    Waiting 3

<code>run_until_complete</code>是一个阻塞（blocking）调用，直到协程运行结束，它才返回。

<code>run_until_complete</code>的参数是一个future，但是这里传给它的却是协程对象，之所以能这样，是因为它在内部做了检查，通过<code>ensure_future</code>函数把协程对象包装（wrap）成了future。可以写得更明显一些：

```ipython
loop.run_until_complete(asyncio.ensure_future(coro(3)))
```

完整的代码：

```ipython
import asyncio

async def coro(wt):
    print('Waiting ' + str(wt))
    await asyncio.sleep(wt)

loop = asyncio.get_event_loop()
loop.run_until_complete(coro(3))
```

    Waiting 3


<a id="org88aa05a"></a>

# 回调

假如协程是一个 IO 的读操作，等它读完数据后希望得到通知，以便下一步数据的处理。可以通过往 future 添加回调来实现。

```ipython
def callback(ft):
    print('Done')


ft = asyncio.ensure_future(coro(3))
ft.add_done_callback(callback)

loop.run_until_complete(ft)
```

    Waiting 3
    Done


<a id="org0d9af90"></a>

# 多个协程

实际项目中，往往有多个协程，同时在一个 loop 里运行。为了把多个协程交给 loop，需要借助<code>asyncio.gather</code>函数。

```ipython
loop.run_until_complete(asyncio.gather(coro(3), coro(2)))
```

    Waiting 2
    Waiting 3

或者先把协程存在列表里：

```ipython
coros = [coro(3), coro(2)]
loop.run_until_complete(asyncio.gather(*coros))
```

    Waiting 2
    Waiting 3

这两个协程是并发运行的，所以等待的时间不是 3 + 2 = 5 秒，而是以耗时较长的那个协程为准。

<code>gather</code>函数也接受 future 对象：

```ipython
fts = [asyncio.ensure_future(coro(3)),
       asyncio.ensure_future(coro(2))]

loop.run_until_complete(asyncio.gather(*fts))
```

    Waiting 3
    Waiting 2

<code>gather</code>起聚合的作用，把多个futures包装成单个future，因为<code>loop.run_until_complete</code>只接受单个future。


<a id="orgb13cf27"></a>

# `run_until_complete` / `run_forever`

通过<code>run_until_complete</code>来运行loop，等到future完成，<code>run_until_complete</code>也就返回了。

改用<code>run_forever</code>：

```ipython
import asyncio

async def coro(wt):
    print('Waiting ' + str(wt))
    await asyncio.sleep(wt)
    print('Done')

loop = asyncio.get_event_loop()
asyncio.ensure_future(coro(3))

loop.run_forever()
```

    Waiting 3
    Done
    # 程序没有退出

3秒之后，future结束，但程序并不会退出。<code>run_forever</code>会一直运行，直到<code>stop</code>被调用，但是不能像下面这样调<code>stop</code>：

```python
loop.run_forever()
loop.stop()
```

<code>run_forever</code>不返回，<code>stop</code>永远也不会被调用。所以，只能在协程中调<code>stop</code>：

```ipython
async def coro(loop, wt):
    print('Waiting ' + str(wt))
    await asyncio.sleep(wt)
    print('Done')
    loop.stop()
```

这样并非没有问题，假如有多个协程在 loop 里运行：

```ipython
asyncio.ensure_future(coro(loop, 3))
asyncio.ensure_future(coro(loop, 2))

loop.run_forever()
```

第二个协程没结束，loop 就停止了——被先结束的那个协程给停掉的。

要解决这个问题，可以用<code>gather</code>把多个协程合并成一个future，并添加回调，然后在回调里再去停止loop。

```ipython
import asyncio
import functools


async def coro(loop, wt):
    print('Waiting ' + str(wt))
    await asyncio.sleep(wt)
    print('Done')

def callback(loop, ft):
    loop.stop()

loop = asyncio.get_event_loop()

ft = asyncio.gather(coro(loop, 3), coro(loop, 2))
ft.add_done_callback(functools.partial(callback, loop))

loop.run_forever()
```

    Waiting 2
    Waiting 3
    Done
    Done

其实这基本上就是<code>run_until_complete</code>的实现了，<code>run_until_complete</code>在内部也是调用<code>run_forever</code>。


<a id="orgda833fd"></a>

# 关闭 loop

上面的例子都没有调用<code>loop.close</code>，loop只要不关闭，就还可以再运行。

```ipython
loop.run_until_complete(coro(loop, 3))
loop.run_until_complete(coro(loop, 2))

loop.close()
```

建议调用<code>loop.close</code>，以彻底清理<code>loop</code>对象防止误用。


<a id="org4c182c8"></a>

# coroutine 和 future

```python
result = await future
result = await coroutine
```

coroutine 是生成器函数，它既可以从外部接受参数，也可以产生结果。使用 coroutine 的好处是可以暂停一个函数，然后稍后恢复执行。在停下的这段时间内，可以切换到其他任务继续执行。

future 更像是一个占位符，其值会在将来被计算出来。例如在等待网络 IO 函数完成时，函数会返回一个容器，future 会在完成时填充该容器。填充完毕后，可以用回调函数来获取实际结果。

<code>Task</code>对象是<code>Future</code>的子类，它将 coroutine 和 future 联系在一起，将 coroutine 封装成一个<code>Future</code>对象。

两种任务启动方法：

```python
tasks = asyncio.gather(
    asyncio.ensure_future(func1()),
    asyncio.ensure_future(func2())
)
loop.run_until_complete(tasks)
```

```python
tasks = [
    asyncio.ensure_future(func1()),
    asyncio.ensure_future(func2())
]
loop.run_until_complete(asyncio.wait(tasks))
```

<code>ensure_future</code>可以将 coroutine 封装成<code>Task</code>类型 。<code>asyncio.gather</code>将 Future 和 coroutine 封装成一个<code>Future</code>对象 。<code>asyncio.wait</code>则本身就是 coroutine 。

关于<code>asyncion.gather()</code>和<code>asyncio.wait()</code>的比较，可具体参考 [StackOverFlow](http://stackoverflow.com/questions/42231161/asyncio-gather-vs-asyncio-wait/42246632#42246632) 。

<code>run_until_complete</code>既可以接收<code>Future</code>对象，也可以是 coroutine 对象。

    BaseEventLoop.run_until_complete(future)

    Run until the Future is done.

    If the argument is a coroutine object, it is wrapped by ensure_future().

    Return the Future’s result, or raise its exception.


<a id="org6cc64a3"></a>

# Task 任务退出

根据官方文档，<code>Task</code>对象只有在以下几种情况，会认为是退出：

    a result / exception are available, or that the future was cancelled

Task 对象的<code>cancel</code>和其父类 Future 略有不同。当调用<code>Task.cancel()</code>后，对应 coroutine 会在事件循环的下一轮中抛出<code>CancelledError</code>异常。使用<code>Future.cancelled()</code>并不能立即返回 True（用来表示任务结束），只有在上述异常被处理任务结束后才算是 cancelled。

结束任务：

```python
for task in asyncio.Task.all_tasks():
    task.cancel()
```

这种方法将所有任务找出并 cancel。

但<CODE>CTRL-C</CODE>也会将事件循环停止，所以有必要重启事件循环，

```python
try:
    loop.run_until_complete(tasks)
except KeyboardInterrupt as e:
    for task in asyncio.Task.all_tasks():
        task.cancel()
        loop.run_forever()  # restart loop
finally:
    loop.close()
```

在每个 Task 中捕获异常是必要的，如果不确定，可以使用<code>asyncio.gather(..., return_exceptions=True)</code>将异常转换为正常的结果返回。


<a id="orgfa4423f"></a>

# Timer

```python
import asyncio


async def timer(wt, cb):
    ft = asyncio.ensure_future(asyncio.sleep(wt))
    ft.add_done_callback(cb)
    print("Waiting {}s ...".format(wt))
    await ft

t = timer(30, lambda ft: print('>>> Done'))
loop = asyncio.get_event_loop()
loop.run_until_complete(t)
```

    Waiting 30s ...
    >>> Done


<a id="org3f10800"></a>

# `async for` / `async with`

<code>async for</code>是异步迭代器语法。为支持异步迭代，异步对象需实现<code>__aiter__</code>方法，异步迭代器需实现<code>__anext__</code>方法，停止迭代需在<code>__anext__</code>方法内抛出<code>StopAsyncIteration</code>异常。

```python
# async_for.py
import random
import asyncio


class AsyncIterable:
    def __init__(self):
        self.count = 0

    async def __aiter__(self):
        return self

    async def __anext__(self):
        if self.count >= 5:
            raise StopAsyncIteration
        data = await self.fetch_data()
        self.count += 1
        return data

    async def fetch_data(self):
        return random.choice(range(10))


async def main():
    async for data in AsyncIterable():
        print(data)


loop = asyncio.get_event_loop()
loop.run_until_complete(main())
```

    0
    8
    3
    9
    5

<code>async with</code>是异步上下文管理器语法，需实现<code>__aenter__</code>和<code>__aexit__</code>方法。

```python
# async_with.py
async def log(msg):
    print(msg)


class AsyncContextManager:
    async def __aenter__(self):
        await log('entering context')

    async def __aexit__(self, exc_type, exc, tb):
        await log('exiting context')


async def coro():
    async with AsyncContextManager():
        print('body')


c = coro()
try:
    c.send(None)
except StopIteration:
    print('finished')
```

    entering context
    body
    exiting context
    finished


<a id="orgca78bf4"></a>

# 示例

```python
import aiohttp
import asyncio


async def get_status(url, id):
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as r:
            print(r.status, id)


tasks = []
for i in range(10):
    ft = asyncio.ensure_future(get_status('https://api.github.com/events', id=i))
    tasks.append(ft)

loop = asyncio.get_event_loop()
loop.run_until_complete(asyncio.wait(tasks))
loop.close()
```

    200 5
    200 0
    200 2
    200 8
    200 1
    200 7
    200 6
    200 3
    200 4
    200 9


<a id="org842917a"></a>

## 示例2

```python
import asyncio
import itertools
import sys


async def spin(msg):
    write, flush = sys.stdout.write, sys.stdout.flush
    for char in itertools.cycle('|/-\\'):
        status = char + ' ' + msg
        write(status)
        flush()
        write('\x08' * len(status))
        try:
            await asyncio.sleep(.1)
        except asyncio.CancelledError:
            break
        write(' ' * len(status) + '\x08' * len(status))


async def slow_function():
    await asyncio.sleep(10)
    return 42


async def supervisor():
    spinner = asyncio.ensure_future(spin('thinking!'))
    print('spinner object:', spinner)
    result = await slow_function()
    spinner.cancel()
    return result


def main():
    loop = asyncio.get_event_loop()
    result = loop.run_until_complete(supervisor())
    loop.close()
    print('Answer:', result)


main()
```


<a id="org1a049ea"></a>

## 示例3

```ipython
import asyncio

import aiohttp
import tqdm


async def print_page(url):
    text = await get(url)
    return text


async def get(*args, **kwargs):
    response = await aiohttp.request('GET', *args, **kwargs)
    return response.status


async def wait_with_progress(coros):
    for f in tqdm.tqdm(asyncio.as_completed(coros), total=len(coros)):
        await asyncio.sleep(2)
        await f


loop = asyncio.get_event_loop()
tasks = []
for i in range(10):
    ft = asyncio.ensure_future(print_page('http://httpbin.org/get'))
    tasks.append(ft)
    loop.run_until_complete(wait_with_progress(tasks))
```

    100%|████████████████████████████████████████████| 10/10 [00:20<00:00,  2.00s/it]


<a id="org8595411"></a>

## 示例4

```ipython
import aiohttp
import asyncio

NUMBERS = range(12)
URL = 'http://httpbin.org/get?a={}'
sema = asyncio.Semaphore(3)


async def fetch_async(a):
    async with aiohttp.request('GET', URL.format(a)) as r:
        data = await r.json()
    return data['args']['a']


async def print_result(a):
    with (await sema):
        r = await fetch_async(a)
        print('fetch({}) = {}'.format(a, r))


loop = asyncio.get_event_loop()
f = asyncio.wait([print_result(num) for num in NUMBERS])
loop.run_until_complete(f)
```

    fetch(5) = 5
    fetch(1) = 1
    fetch(10) = 10
    fetch(8) = 8
    fetch(9) = 9
    fetch(3) = 3
    fetch(7) = 7
    fetch(11) = 11
    fetch(2) = 2
    fetch(6) = 6
    fetch(4) = 4
    fetch(0) = 0
