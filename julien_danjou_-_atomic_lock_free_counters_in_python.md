At Datadog, we're really into metrics. We love them, we store them, but we also generate them. To do that, you need to juggle with integers that are incremented, also known as counters.

While having an integer that changes its value sounds dull, it might not be without some surprises in certain circumstances. Let's dive in.

## The Straightforward Implementation
```
class SingleThreadCounter(object):
	def __init__(self):
    	self.value = 0
        
    def increment(self):
        self.value += 1
```
Pretty easy, right?

Well, not so fast, buddy. As the class name implies, this works fine with a single-threaded application. Let's take a look at the instructions in the `increment` method:
```
>>> import dis
>>> dis.dis("self.value += 1")
  1           0 LOAD_NAME                0 (self)
              2 DUP_TOP
              4 LOAD_ATTR                1 (value)
              6 LOAD_CONST               0 (1)
              8 INPLACE_ADD
             10 ROT_TWO
             12 STORE_ATTR               1 (value)
             14 LOAD_CONST               1 (None)
             16 RETURN_VALUE
```
The `self.value +=1` line of code generates 8 different operations for Python. Operations that could be interrupted at any time in their flow to switch to a different thread that could also increment the counter.

Indeed, the `+=` operation is not atomic: one needs to do a `LOAD_ATTR` to read the current value of the counter, then an `INPLACE_ADD` to add 1, to finally `STORE_ATTR` to store the final result in the value attribute.

If another thread executes the same code at the same time, you could end up with adding 1 to an old value:
```
Thread-1 reads the value as 23
Thread-1 adds 1 to 23 and get 24
Thread-2 reads the value as 23
Thread-1 stores 24 in value
Thread-2 adds 1 to 23
Thread-2 stores 24 in value
```
Boom. Your Counter class is not thread-safe. 😭

## The Thread-Safe Implementation
To make this thread-safe, a _lock_ is necessary. We need a lock each time we want to increment the value, so we are sure the increments are done serially.
```
import threading

class FastReadCounter(object):
    def __init__(self):
        self.value = 0
        self._lock = threading.Lock()
        
    def increment(self):
        with self._lock:
            self.value += 1
```
This implementation is thread-safe. There is no way for multiple threads to increment the value at the same time, so there's no way that an increment is lost.

The only downside of this counter implementation is that you need to lock the counter each time you need to increment. There might be much contention around this lock if you have many threads updating the counter.

On the other hand, if it's barely updated and often read, this is an excellent implementation of a thread-safe counter.

## A Fast Write Implementation
There's a way to implement a thread-safe counter in Python that does not need to be locked on write. It's a trick that should only work on CPython because of the _Global Interpreter Lock_.

While everybody is unhappy with it, this time, the GIL is going to help us. When a C function is executed and does not do any I/O, it cannot be interrupted by any other thread. It turns out there's a counter-like class implemented in Python: [itertools.count](https://docs.python.org/3/library/itertools.html#itertools.count).

We can use this `count` class as our advantage by avoiding the need to use a lock when incrementing the counter.

If you read the documentation for `itertools.count`, you'll notice that there's no way to read the current value of the counter. This is tricky, and this is where we'll need to use a lock to bypass this limitation. Here's the code:
```
import itertools
import threading

class FastWriteCounter(object):
    def __init__(self):
        self._number_of_read = 0
        self._counter = itertools.count()
        self._read_lock = threading.Lock()

    def increment(self):
        next(self._counter)

    def value(self):
        with self._read_lock:
            value = next(self._counter) - self._number_of_read
            self._number_of_read += 1
        return value
```

The `increment` code is quite simple in this case: the counter is just incremented without any lock. The GIL protects concurrent access to the internal data structure in C, so there's no need for us to lock anything.

On the other hand, Python does not provide any way to read the value of an `itertools.count` object. We need to use a small trick to get the current value. The `value` method increments the counter and then gets the value while subtracting the number of times the counter has been read (and therefore incremented for nothing).

This counter is, therefore, lock-free for writing, but not for reading. The opposite of our previous implementation

## Measuring Performance
After writing all of this code, I wanted to make sure how the different implementations impacted speed. Using the [timeit](https://docs.python.org/3/library/timeit.html) module and my fancy laptop, I've measured the performance of reading and writing to this counter.

| Operation |	SingleThreadCounter	| FastReadCounter |	FastWriteCounter |
| --- | --- | --- | --- |
| increment |	176 ns	| 390 ns |	169 ns |
| value |	26 ns |	26 ns |	529 ns |

I'm glad that the performance measurements in practice match the theory 😅. Both `SingleThreadCounter` and `FastReadCounter` have the same performance for reading. Since they use a simple variable read, it makes absolute sense.

The same goes for `SingleThreadCounter` and `FastWriteCounter`, which have the same performance for incrementing the counter. Again they're using the same kind of lock-free code to add 1 to an integer, making the code fast.

## Conclusion
It's pretty obvious, but if you're using a single-threaded application and do not have to care about concurrent access, you should stick to using a simple incremented integer.

For fun, I've published a Python package named [fastcounter](https://pypi.org/project/fastcounter/) that provides those classes. The sources are [available on GitHub](https://github.com/jd/fastcounter). Enjoy!
