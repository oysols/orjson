# orjson

orjson is a fast, correct JSON library for Python. It
[benchmarks](https://github.com/ijl/orjson#performance) as the fastest Python
library for JSON and is more correct than the standard json library or other
third-party libraries. It serializes
[dataclass](https://github.com/ijl/orjson#dataclass),
[datetime](https://github.com/ijl/orjson#datetime),
[numpy](https://github.com/ijl/orjson#numpy), and
[UUID](https://github.com/ijl/orjson#uuid) instances natively.

Its features and drawbacks compared to other Python JSON libraries:

* serializes `dataclass` instances 40-50x as fast as other libraries
* serializes `datetime`, `date`, and `time` instances to RFC 3339 format,
e.g., "1970-01-01T00:00:00+00:00"
* serializes `numpy.ndarray` instances 4-12x as fast with 0.3x the memory
usage of other libraries
* pretty prints 10x to 20x as fast as the standard library
* serializes to `bytes` rather than `str`, i.e., is not a drop-in replacement
* serializes `str` without escaping unicode to ASCII, e.g., "好" rather than
"\\\u597d"
* serializes `float` 10x as fast and deserializes twice as fast as other
libraries
* serializes arbitrary types using a `default` hook
* has strict UTF-8 conformance, more correct than the standard library
* has strict JSON conformance in not supporting Nan/Infinity/-Infinity
* has an option for strict JSON conformance on 53-bit integers with default
support for 64-bit
* does not support subclasses by default, requiring use of `default` hook
* does not provide `load()` or `dump()` functions for reading from/writing to
file-like objects

orjson supports CPython 3.6, 3.7, 3.8, and 3.9. It distributes x86_64/amd64
and aarch64/armv8 wheels for Linux. It distributes x86_64/amd64 wheels for
macOS and Windows. orjson does not support PyPy.

orjson is licensed under both the Apache 2.0 and MIT licenses. The
repository and issue tracker is
[github.com/ijl/orjson](https://github.com/ijl/orjson), and patches may be
submitted there. There is a
[CHANGELOG](https://github.com/ijl/orjson/blob/master/CHANGELOG.md)
available in the repository.

1. [Usage](https://github.com/ijl/orjson#usage)
    1. [Install](https://github.com/ijl/orjson#install)
    1. [Quickstart](https://github.com/ijl/orjson#quickstart)
    3. [Serialize](https://github.com/ijl/orjson#serialize)
        1. [default](https://github.com/ijl/orjson#default)
        2. [option](https://github.com/ijl/orjson#option)
    4. [Deserialize](https://github.com/ijl/orjson#deserialize)
2. [Types](https://github.com/ijl/orjson#types)
    1. [dataclass](https://github.com/ijl/orjson#dataclass)
    2. [datetime](https://github.com/ijl/orjson#datetime)
    3. [enum](https://github.com/ijl/orjson#enum)
    4. [float](https://github.com/ijl/orjson#float)
    5. [int](https://github.com/ijl/orjson#int)
    6. [numpy](https://github.com/ijl/orjson#numpy)
    7. [str](https://github.com/ijl/orjson#str)
    8. [uuid](https://github.com/ijl/orjson#uuid)
3. [Testing](https://github.com/ijl/orjson#testing)
4. [Performance](https://github.com/ijl/orjson#performance)
    1. [Latency](https://github.com/ijl/orjson#latency)
    2. [Memory](https://github.com/ijl/orjson#memory)
    3. [Reproducing](https://github.com/ijl/orjson#reproducing)
5. [Packaging](https://github.com/ijl/orjson#packaging)
6. [License](https://github.com/ijl/orjson#license)

## Usage

### Install

To install a wheel from PyPI:

```sh
pip install --upgrade orjson
```

To depend on orjson in a project:

```txt
orjson>=2.6,<3
```

To build a wheel, see [packaging](https://github.com/ijl/orjson#packaging).


### Quickstart

This is an example of serializing, with options specified, and deserializing:

```python
>>> import orjson, datetime, numpy
>>> data = {
    "type": "job",
    "created_at": datetime.datetime(1970, 1, 1),
    "status": "🆗",
    "payload": numpy.array([[1, 2], [3, 4]]),
}
>>> orjson.dumps(data, option=orjson.OPT_NAIVE_UTC | orjson.OPT_SERIALIZE_NUMPY)
b'{"type":"job","created_at":"1970-01-01T00:00:00+00:00","status":"\xf0\x9f\x86\x97","payload":[[1,2],[3,4]]}'
>>> orjson.loads(_)
{'type': 'job', 'created_at': '1970-01-01T00:00:00+00:00', 'status': '🆗', 'payload': [[1, 2], [3, 4]]}
```

### Serialize

```python
def dumps(
    __obj: Any,
    default: Optional[Callable[[Any], Any]] = ...,
    option: Optional[int] = ...,
) -> bytes: ...
```

`dumps()` serializes Python objects to JSON.

It natively serializes
`str`, `dict`, `list`, `tuple`, `int`, `float`, `bool`,
`dataclasses.dataclass`, `typing.TypedDict`, `datetime.datetime`,
`datetime.date`, `datetime.time`, `uuid.UUID`, `numpy.ndarray`, and
`None` instances. It supports arbitrary types through `default`.
It does not serialize subclasses of
supported types natively, with the exception of `dataclasses.dataclass`
subclasses.

The output is a `bytes` object containing UTF-8.

It raises `JSONEncodeError` on an unsupported type. This exception message
describes the invalid object.

It raises `JSONEncodeError` on a `str` that contains invalid UTF-8.

It raises `JSONEncodeError` on an integer that exceeds 64 bits by default or,
with `OPT_STRICT_INTEGER`, 53 bits.

It raises `JSONEncodeError` if a `dict` has a key of a type other than `str`,
unless `OPT_NON_STR_KEYS` is specified.

It raises `JSONEncodeError` if the output of `default` recurses to handling by
`default` more than 254 levels deep.

It raises `JSONEncodeError` on circular references.

It raises `JSONEncodeError`  if a `tzinfo` on a datetime object is
unsupported.

`JSONEncodeError` is a subclass of `TypeError`. This is for compatibility
with the standard library.

#### default

To serialize a subclass or arbitrary types, specify `default` as a
callable that returns a supported type. `default` may be a function,
lambda, or callable class instance. To specify that a type was not
handled by `default`, raise an exception such as `TypeError`.

```python
>>> import orjson, decimal
>>>
def default(obj):
    if isinstance(obj, decimal.Decimal):
        return str(obj)
    raise TypeError

>>> orjson.dumps(decimal.Decimal("0.0842389659712649442845"))
JSONEncodeError: Type is not JSON serializable: decimal.Decimal
>>> orjson.dumps(decimal.Decimal("0.0842389659712649442845"), default=default)
b'"0.0842389659712649442845"'
>>> orjson.dumps({1, 2}, default=default)
orjson.JSONEncodeError: Type is not JSON serializable: set
```

The `default` callable may return an object that itself
must be handled by `default` up to 254 times before an exception
is raised.

It is important that `default` raise an exception if a type cannot be handled.
Python otherwise implicitly returns `None`, which appears to the caller
like a legitimate value and is serialized:

```python
>>> import orjson, json, rapidjson
>>>
def default(obj):
    if isinstance(obj, decimal.Decimal):
        return str(obj)

>>> orjson.dumps({"set":{1, 2}}, default=default)
b'{"set":null}'
>>> json.dumps({"set":{1, 2}}, default=default)
'{"set":null}'
>>> rapidjson.dumps({"set":{1, 2}}, default=default)
'{"set":null}'
```

#### option

To modify how data is serialized, specify `option`. Each `option` is an integer
constant in `orjson`. To specify multiple options, mask them together, e.g.,
`option=orjson.OPT_STRICT_INTEGER | orjson.OPT_NAIVE_UTC`.

##### OPT_INDENT_2

Pretty-print output with an indent of two spaces. This is equivalent to
`indent=2` in the standard library. Pretty printing is slower and the output
larger. orjson is the fastest compared library at pretty printing and has
much less of a slowdown to pretty print than the standard library does. This
option is compatible with all other options.

```python
>>> import orjson
>>> orjson.dumps({"a": "b", "c": {"d": True}, "e": [1, 2]})
b'{"a":"b","c":{"d":true},"e":[1,2]}'
>>> orjson.dumps(
    {"a": "b", "c": {"d": True}, "e": [1, 2]},
    option=orjson.OPT_INDENT_2
)
b'{\n  "a": "b",\n  "c": {\n    "d": true\n  },\n  "e": [\n    1,\n    2\n  ]\n}'
```

If displayed, the indentation and linebreaks appear like this:

```json
{
  "a": "b",
  "c": {
    "d": true
  },
  "e": [
    1,
    2
  ]
}
```

This measures serializing the github.json fixture as compact (52KiB) or
pretty (64KiB):

| Library    |   compact (ms) | pretty (ms)   | vs. orjson   |
|------------|----------------|---------------|--------------|
| orjson     |           0.07 | 0.07          | 1.0          |
| ujson      |           0.16 | 0.17          | 2.4          |
| rapidjson  |           0.29 |               |              |
| simplejson |           0.48 | 1.69          | 22.9         |
| json       |           0.35 | 1.28          | 17.4         |

This measures serializing the citm_catalog.json fixture, more of a worst
case due to the amount of nesting and newlines, as compact (489KiB) or
pretty (1.1MiB):

| Library    |   compact (ms) | pretty (ms)   | vs. orjson   |
|------------|----------------|---------------|--------------|
| orjson     |           1.32 | 2.49          | 1.0          |
| ujson      |           3.67 | 5.23          | 2.1          |
| rapidjson  |           3.67 |               |              |
| simplejson |          13.13 | 78.74         | 31.7         |
| json       |           7.87 | 59.22         | 23.8         |

rapidjson is blank because it does not support pretty printing. This can be
reproduced using the `pyindent` script.

##### OPT_NAIVE_UTC

Serialize `datetime.datetime` objects without a `tzinfo` as UTC. This
has no effect on `datetime.datetime` objects that have `tzinfo` set.

```python
>>> import orjson, datetime
>>> orjson.dumps(
        datetime.datetime(1970, 1, 1, 0, 0, 0),
    )
b'"1970-01-01T00:00:00"'
>>> orjson.dumps(
        datetime.datetime(1970, 1, 1, 0, 0, 0),
        option=orjson.OPT_NAIVE_UTC,
    )
b'"1970-01-01T00:00:00+00:00"'
```

##### OPT_NON_STR_KEYS

Serialize `dict` keys of type other than `str`. This allows `dict` keys
to be one of `str`, `int`, `float`, `bool`, `None`, `datetime.datetime`,
`datetime.date`, `datetime.time`, `enum.Enum`, and `uuid.UUID`. For comparison,
the standard library serializes `str`, `int`, `float`, `bool` or `None` by
default. orjson benchmarks as being faster at serializing non-`str` keys
than other libraries. This option is slower for `str` keys than the default
and is not recommended generally.

```python
>>> import orjson, datetime, uuid
>>> orjson.dumps(
        {uuid.UUID("7202d115-7ff3-4c81-a7c1-2a1f067b1ece"): [1, 2, 3]},
        option=orjson.OPT_NON_STR_KEYS,
    )
b'{"7202d115-7ff3-4c81-a7c1-2a1f067b1ece":[1,2,3]}'
>>> orjson.dumps(
        {datetime.datetime(1970, 1, 1, 0, 0, 0): [1, 2, 3]},
        option=orjson.OPT_NON_STR_KEYS | orjson.OPT_NAIVE_UTC,
    )
b'{"1970-01-01T00:00:00+00:00":[1,2,3]}'
```

These types are generally serialized how they would be as
values, e.g., `datetime.datetime` is still an RFC 3339 string and respects
options affecting it. The exception is that `int` serialization does not
respect `OPT_STRICT_INTEGER`.

This option has the risk of creating duplicate keys. This is because non-`str`
objects may serialize to the same `str` as an existing key, e.g.,
`{"1": true, 1: false}`. The last key to be inserted to the `dict` will be
serialized last and a JSON deserializer will presumably take the last
occurrence of a key (in the above, `false`). The first value will be lost.

This option is compatible with `orjson.OPT_SORT_KEYS`. If sorting is used,
note the sort is unstable and will be unpredictable for duplicate keys.

```python
>>> import orjson, datetime
>>> orjson.dumps(
    {"other": 1, datetime.date(1970, 1, 5): 2, datetime.date(1970, 1, 3): 3},
    option=orjson.OPT_NON_STR_KEYS | orjson.OPT_SORT_KEYS
)
b'{"1970-01-03":3,"1970-01-05":2,"other":1}'
```

This measures serializing 589KiB of JSON comprising a `list` of 100 `dict`
in which each `dict` has both 365 randomly-sorted `int` keys representing epoch
timestamps as well as one `str` key and the value for each key is a
single integer. In "str keys", the keys were converted to `str` before
serialization, and orjson still specifes `option=orjson.OPT_NON_STR_KEYS`
(which is always somewhat slower).

| Library    |   str keys (ms) | int keys (ms)   | int keys sorted (ms)   |
|------------|-----------------|-----------------|------------------------|
| orjson     |            1.97 | 2.24            | 6.50                   |
| ujson      |            2.82 | 5.32            |                        |
| rapidjson  |            4.47 |                 |                        |
| simplejson |            9.42 | 11.77           | 21.52                  |
| json       |            6.32 | 8.05            |                        |

ujson is blank for sorting because it segfaults. json is blank because it
raises `TypeError` on attempting to sort before converting all keys to `str`.
rapidjson is blank because it does not support non-`str` keys. This can
be reproduced using the `pynonstr` script.

##### OPT_OMIT_MICROSECONDS

Do not serialize the `microsecond` field on `datetime.datetime` and
`datetime.time` instances.

```python
>>> import orjson, datetime
>>> orjson.dumps(
        datetime.datetime(1970, 1, 1, 0, 0, 0, 1),
    )
b'"1970-01-01T00:00:00.000001"'
>>> orjson.dumps(
        datetime.datetime(1970, 1, 1, 0, 0, 0, 1),
        option=orjson.OPT_OMIT_MICROSECONDS,
    )
b'"1970-01-01T00:00:00"'
```

##### OPT_SERIALIZE_DATACLASS

Serialize `dataclasses.dataclass` instances. For more, see
[dataclass](https://github.com/ijl/orjson#dataclass).

##### OPT_SERIALIZE_NUMPY

Serialize `numpy.ndarray` instances. For more, see
[numpy](https://github.com/ijl/orjson#numpy).

##### OPT_SERIALIZE_UUID

Serialize `uuid.UUID` instances. For more, see
[UUID](https://github.com/ijl/orjson#UUID).

##### OPT_SORT_KEYS

Serialize `dict` keys in sorted order. The default is to serialize in an
unspecified order. This is equivalent to `sort_keys=True` in the standard
library.

This can be used to ensure the order is deterministic for hashing or tests.
It has a substantial performance penalty and is not recommended in general.

```python
>>> import orjson
>>> orjson.dumps({"b": 1, "c": 2, "a": 3})
b'{"b":1,"c":2,"a":3}'
>>> orjson.dumps({"b": 1, "c": 2, "a": 3}, option=orjson.OPT_SORT_KEYS)
b'{"a":3,"b":1,"c":2}'
```

This measures serializing the twitter.json fixture unsorted and sorted:

| Library    |   unsorted (ms) |   sorted (ms) |   vs. orjson |
|------------|-----------------|---------------|--------------|
| orjson     |            0.5  |          0.92 |          1   |
| ujson      |            1.61 |          2.48 |          2.7 |
| rapidjson  |            2.17 |          2.89 |          3.2 |
| simplejson |            3.56 |          5.13 |          5.6 |
| json       |            3.59 |          4.59 |          5   |

The benchmark can be reproduced using the `pysort` script.

The sorting is not collation/locale-aware:

```python
>>> import orjson
>>> orjson.dumps({"a": 1, "ä": 2, "A": 3}, option=orjson.OPT_SORT_KEYS)
b'{"A":3,"a":1,"\xc3\xa4":2}'
```

This is the same sorting behavior as the standard library, rapidjson,
simplejson, and ujson.

`dataclass` also serialize as maps but this has no effect on them.

##### OPT_STRICT_INTEGER

Enforce 53-bit limit on integers. The limit is otherwise 64 bits, the same as
the Python standard library. For more, see [int](https://github.com/ijl/orjson#int).

##### OPT_UTC_Z

Serialize a UTC timezone on `datetime.datetime` instances as `Z` instead
of `+00:00`.

```python
>>> import orjson, datetime
>>> orjson.dumps(
        datetime.datetime(1970, 1, 1, 0, 0, 0, tzinfo=datetime.timezone.utc),
    )
b'"1970-01-01T00:00:00+00:00"'
>>> orjson.dumps(
        datetime.datetime(1970, 1, 1, 0, 0, 0, tzinfo=datetime.timezone.utc),
        option=orjson.OPT_UTC_Z
    )
b'"1970-01-01T00:00:00Z"'
```

### Deserialize

```python
def loads(__obj: Union[bytes, bytearray, str]) -> Any: ...
```

`loads()` deserializes JSON to Python objects. It deserializes to `dict`,
`list`, `int`, `float`, `str`, `bool`, and `None` objects.

`bytes`, `bytearray`, and `str` input are accepted. If the input exists as
`bytes` (was read directly from a source), it is recommended to
pass `bytes`. This has lower memory usage and lower latency.

The input must be valid UTF-8.

orjson maintains a cache of map keys for the duration of the process. This
causes a net reduction in memory usage by avoiding duplicate strings. The
keys must be at most 64 chars to be cached and 512 entries are stored.

It raises `JSONDecodeError` if given an invalid type or invalid
JSON. This includes if the input contains `NaN`, `Infinity`, or `-Infinity`,
which the standard library allows, but is not valid JSON.

`JSONDecodeError` is a subclass of `json.JSONDecodeError` and `ValueError`.
This is for compatibility with the standard library.

## Types

### dataclass

orjson serializes instances of `dataclasses.dataclass` natively. It serializes
instances 40-50x as fast as other libraries and avoids a severe slowdown seen
in other libraries compared to serializing `dict`. To serialize
instances, specify `option=orjson.OPT_SERIALIZE_DATACLASS`.

It is supported to pass all variants of dataclasses, including dataclasses
using `__slots__`, frozen dataclasses, those with optional or default
attributes, and subclasses. There is a performance benefit to not
using `__slots__`.

| Library    | dict (ms)   | dataclass (ms)   | vs. orjson   |
|------------|-------------|------------------|--------------|
| orjson     | 1.40        | 1.60             | 1            |
| ujson      |             |                  |              |
| rapidjson  | 3.64        | 68.48            | 42           |
| simplejson | 14.21       | 92.18            | 57           |
| json       | 13.28       | 94.90            | 59           |

This measures serializing 555KiB of JSON, orjson natively and other libraries
using `default` to serialize the output of `dataclasses.asdict()`. This can be
reproduced using the `pydataclass` script.

Dataclasses are serialized as maps, with every attribute serialized and in
the order given on class definition:

```python
>>> import dataclasses, orjson, typing

@dataclasses.dataclass
class Member:
    id: int
    active: bool = dataclasses.field(default=False)

@dataclasses.dataclass
class Object:
    id: int
    name: str
    members: typing.List[Member]

>>> orjson.dumps(
        Object(1, "a", [Member(1, True), Member(2)]),
        option=orjson.OPT_SERIALIZE_DATACLASS,
    )
b'{"id":1,"name":"a","members":[{"id":1,"active":true},{"id":2,"active":false}]}'
```
Users may wish to control how dataclass instances are serialized, e.g.,
to not serialize an attribute or to change the name of an
attribute when serialized. orjson may implement support using the
metadata mapping on `field` attributes,
e.g., `field(metadata={"json_serialize": False})`, if use cases are clear.

### datetime

orjson serializes `datetime.datetime` objects to
[RFC 3339](https://tools.ietf.org/html/rfc3339) format,
e.g., "1970-01-01T00:00:00+00:00". This is a subset of ISO 8601 and
compatible with `isoformat()` in the standard library.

```python
>>> import orjson, datetime, pendulum
>>> orjson.dumps(
    datetime.datetime(2018, 12, 1, 2, 3, 4, 9, tzinfo=pendulum.timezone('Australia/Adelaide'))
)
b'"2018-12-01T02:03:04.000009+10:30"'
>>> orjson.dumps(
    datetime.datetime.fromtimestamp(4123518902).replace(tzinfo=datetime.timezone.utc)
)
b'"2100-09-01T21:55:02+00:00"'
>>> orjson.dumps(
    datetime.datetime.fromtimestamp(4123518902)
)
b'"2100-09-01T21:55:02"'
```

`datetime.datetime` supports instances with a `tzinfo` that is `None`,
`datetime.timezone.utc` or a timezone instance from
the `pendulum`, `pytz`, or `dateutil`/`arrow` libraries.

`datetime.time` objects must not have a `tzinfo`.

```python
>>> import orjson, datetime
>>> orjson.dumps(datetime.time(12, 0, 15, 290))
b'"12:00:15.000290"'
```

`datetime.date` objects will always serialize.

```python
>>> import orjson, datetime
>>> orjson.dumps(datetime.date(1900, 1, 2))
b'"1900-01-02"'
```

Errors with `tzinfo` result in `JSONEncodeError` being raised.

It is faster to have orjson serialize datetime objects than to do so
before calling `dumps()`. If using an unsupported type such as
`pendulum.datetime`, use `default`.

### enum

orjson serializes enums natively. Options apply to their values.

```python
>>> import enum, datetime, orjson
>>>
class DatetimeEnum(enum.Enum):
    EPOCH = datetime.datetime(1970, 1, 1, 0, 0, 0)
>>> orjson.dumps(DatetimeEnum.EPOCH)
b'"1970-01-01T00:00:00"'
>>> orjson.dumps(DatetimeEnum.EPOCH, option=orjson.OPT_NAIVE_UTC)
b'"1970-01-01T00:00:00+00:00"'
```

Enums with members that are not supported types can be serialized using
`default`:

```python
>>> import enum, orjson
>>>
class Custom:
    def __init__(self, val):
        self.val = val

def default(obj):
    if isinstance(obj, Custom):
        return obj.val
    raise TypeError

class CustomEnum(enum.Enum):
    ONE = Custom(1)

>>> orjson.dumps(CustomEnum.ONE, default=default)
b'1'
```

### float

orjson serializes and deserializes double precision floats with no loss of
precision and consistent rounding. The same behavior is observed in rapidjson,
simplejson, and json. ujson 1.35 was inaccurate in both serialization and
deserialization, i.e., it modifies the data, and the recent 2.0 release is
accurate.

`orjson.dumps()` serializes Nan, Infinity, and -Infinity, which are not
compliant JSON, as `null`:

```python
>>> import orjson, ujson, rapidjson, json
>>> orjson.dumps([float("NaN"), float("Infinity"), float("-Infinity")])
b'[null,null,null]'
>>> ujson.dumps([float("NaN"), float("Infinity"), float("-Infinity")])
OverflowError: Invalid Inf value when encoding double
>>> rapidjson.dumps([float("NaN"), float("Infinity"), float("-Infinity")])
'[NaN,Infinity,-Infinity]'
>>> json.dumps([float("NaN"), float("Infinity"), float("-Infinity")])
'[NaN, Infinity, -Infinity]'
```

### int

JSON only requires that implementations accept integers with 53-bit precision.
orjson will, by default, serialize 64-bit integers. This is compatible with
the Python standard library and other non-browser implementations. For
transmitting JSON to a web browser or other strict implementations, `dumps()`
can be configured to raise a `JSONEncodeError` on values exceeding the
53-bit range.

```python
>>> import orjson
>>> orjson.dumps(9007199254740992)
b'9007199254740992'
>>> orjson.dumps(9007199254740992, option=orjson.OPT_STRICT_INTEGER)
JSONEncodeError: Integer exceeds 53-bit range
>>> orjson.dumps(-9007199254740992, option=orjson.OPT_STRICT_INTEGER)
JSONEncodeError: Integer exceeds 53-bit range
```

### numpy

orjson natively serializes `numpy.ndarray` instances. Arrays may have a
`dtype` of `numpy.bool`, `numpy.float32`, `numpy.float64`, `numpy.int32`,
`numpy.int64`, `numpy.uint32`, `numpy.uint64`, `numpy.uintp`, or `numpy.intp`.
orjson is faster than all compared libraries at serializing
numpy instances. Serializing numpy data requires specifying
`option=orjson.OPT_SERIALIZE_NUMPY`.

```python
>>> import orjson, numpy
>>> orjson.dumps(
        numpy.array([[1, 2, 3], [4, 5, 6]]),
        option=orjson.OPT_SERIALIZE_NUMPY,
)
b'[[1,2,3],[4,5,6]]'
```

The array must be a contiguous C array (`C_CONTIGUOUS`) and one of the
supported datatypes. Individual items (e.g., `numpy.float64(1)`) are
not supported.

If an array is not a contiguous C array or contains an supported datatype,
orjson falls through to `default`. In `default`, `obj.tolist()` can be
specified. If an array is malformed, which is not expected,
`orjson.JSONEncodeError` is raised.

This measures serializing 92MiB of JSON from an `numpy.ndarray` with
dimensions of `(50000, 100)` and `numpy.float64` values:

| Library    | Latency (ms)   | RSS diff (MiB)   | vs. orjson   |
|------------|----------------|------------------|--------------|
| orjson     | 302            | 99               | 1.0          |
| ujson      |                |                  |              |
| rapidjson  | 3,620          | 310              | 12.0         |
| simplejson | 3,596          | 297              | 11.9         |
| json       | 3,410          | 298              | 11.3         |

This measures serializing 100MiB of JSON from an `numpy.ndarray` with
dimensions of `(100000, 100)` and `numpy.int32` values:

| Library    | Latency (ms)   | RSS diff (MiB)   | vs. orjson   |
|------------|----------------|------------------|--------------|
| orjson     | 191            | 118              | 1.0          |
| ujson      |                |                  |              |
| rapidjson  | 1,808          | 553              | 9.5          |
| simplejson | 1,796          | 506              | 9.4          |
| json       | 1,590          | 506              | 8.3          |

This measures serializing 105MiB of JSON from an `numpy.ndarray` with
dimensions of `(100000, 200)` and `numpy.bool` values:

| Library    | Latency (ms)   | RSS diff (MiB)   | vs. orjson   |
|------------|----------------|------------------|--------------|
| orjson     | 211            | 123              | 1.0          |
| ujson      |                |                  |              |
| rapidjson  | 919            | 346              | 4.3          |
| simplejson | 1,239          | 367              | 5.9          |
| json       | 1,243          | 367              | 5.9          |

In these benchmarks, orjson serializes natively, ujson is blank because it
does not support a `default` parameter, and the other libraries serialize
`ndarray.tolist()` via `default`. The RSS column measures peak memory
usage during serialization. This can be reproduced using the `pynumpy` script.

orjson does not have an installation or compilation dependency on numpy. The
implementation is independent, reading `numpy.ndarray` using
`PyArrayInterface`.

### str

orjson is strict about UTF-8 conformance. This is stricter than the standard
library's json module, which will serialize and deserialize UTF-16 surrogates,
e.g., "\ud800", that are invalid UTF-8.

If `orjson.dumps()` is given a `str` that does not contain valid UTF-8,
`orjson.JSONEncodeError` is raised. If `loads()` receives invalid UTF-8,
`orjson.JSONDecodeError` is raised.

orjson and rapidjson are the only compared JSON libraries to consistently
error on bad input.

```python
>>> import orjson, ujson, rapidjson, json
>>> orjson.dumps('\ud800')
JSONEncodeError: str is not valid UTF-8: surrogates not allowed
>>> ujson.dumps('\ud800')
UnicodeEncodeError: 'utf-8' codec ...
>>> rapidjson.dumps('\ud800')
UnicodeEncodeError: 'utf-8' codec ...
>>> json.dumps('\ud800')
'"\\ud800"'
>>> orjson.loads('"\\ud800"')
JSONDecodeError: unexpected end of hex escape at line 1 column 8: line 1 column 1 (char 0)
>>> ujson.loads('"\\ud800"')
''
>>> rapidjson.loads('"\\ud800"')
ValueError: Parse error at offset 1: The surrogate pair in string is invalid.
>>> json.loads('"\\ud800"')
'\ud800'
```

To make a best effort at deserializing bad input, first decode `bytes` using
the `replace` or `lossy` argument for `errors`:

```python
>>> import orjson
>>> orjson.loads(b'"\xed\xa0\x80"')
JSONDecodeError: str is not valid UTF-8: surrogates not allowed
>>> orjson.loads(b'"\xed\xa0\x80"'.decode("utf-8", "replace"))
'���'
```

### uuid

orjson serializes `uuid.UUID` instances to
[RFC 4122](https://tools.ietf.org/html/rfc4122) format, e.g.,
"f81d4fae-7dec-11d0-a765-00a0c91e6bf6". This requires specifying
`option=orjson.OPT_SERIALIZE_UUID`.

``` python
>>> import orjson, uuid
>>> orjson.dumps(
    uuid.UUID('f81d4fae-7dec-11d0-a765-00a0c91e6bf6'),
    option=orjson.OPT_SERIALIZE_UUID,
)
b'"f81d4fae-7dec-11d0-a765-00a0c91e6bf6"'
>>> orjson.dumps(
    uuid.uuid5(uuid.NAMESPACE_DNS, "python.org"),
    option=orjson.OPT_SERIALIZE_UUID,
)
b'"886313e1-3b8a-5372-9b90-0c9aee199e5d"'
```

## Testing

The library has comprehensive tests. There are tests against fixtures in the
[JSONTestSuite](https://github.com/nst/JSONTestSuite) and
[nativejson-benchmark](https://github.com/miloyip/nativejson-benchmark)
repositories. It is tested to not crash against the
[Big List of Naughty Strings](https://github.com/minimaxir/big-list-of-naughty-strings).
It is tested to not leak memory. It is tested to not crash
against and not accept invalid UTF-8. There are integration tests
exercising the library's use in web servers (gunicorn using multiprocess/forked
workers) and when
multithreaded. It also uses some tests from the ultrajson library.

## Performance

Serialization and deserialization performance of orjson is better than
ultrajson, rapidjson, simplejson, or json. The benchmarks are done on
fixtures of real data:

* twitter.json, 631.5KiB, results of a search on Twitter for "一", containing
CJK strings, dictionaries of strings and arrays of dictionaries, indented.

* github.json, 55.8KiB, a GitHub activity feed, containing dictionaries of
strings and arrays of dictionaries, not indented.

* citm_catalog.json, 1.7MiB, concert data, containing nested dictionaries of
strings and arrays of integers, indented.

* canada.json, 2.2MiB, coordinates of the Canadian border in GeoJSON
format, containing floats and arrays, indented.

### Latency

![alt text](doc/twitter_serialization.png "twitter.json serialization")
![alt text](doc/twitter_deserialization.png "twitter.json deserialization")
![alt text](doc/github_serialization.png "github.json serialization")
![alt text](doc/github_deserialization.png "github.json deserialization")
![alt text](doc/citm_catalog_serialization.png "citm_catalog.json serialization")
![alt text](doc/citm_catalog_deserialization.png "citm_catalog.json deserialization")
![alt text](doc/canada_serialization.png "canada.json serialization")
![alt text](doc/canada_deserialization.png "canada.json deserialization")

#### twitter.json serialization

| Library    |   Median latency (milliseconds) |   Operations per second |   Relative (latency) |
|------------|---------------------------------|-------------------------|----------------------|
| orjson     |                            0.66 |                  1524.6 |                 1    |
| ujson      |                            2.04 |                   489.4 |                 3.11 |
| rapidjson  |                            2.44 |                   410.2 |                 3.71 |
| simplejson |                            3.22 |                   309.7 |                 4.91 |
| json       |                            3.24 |                   309.1 |                 4.93 |

#### twitter.json deserialization

| Library    |   Median latency (milliseconds) |   Operations per second |   Relative (latency) |
|------------|---------------------------------|-------------------------|----------------------|
| orjson     |                            2.46 |                   406.9 |                 1    |
| ujson      |                            2.9  |                   344.6 |                 1.18 |
| rapidjson  |                            3.74 |                   267.3 |                 1.52 |
| simplejson |                            3.64 |                   276.5 |                 1.48 |
| json       |                            4.21 |                   238.9 |                 1.71 |

#### github.json serialization

| Library    |   Median latency (milliseconds) |   Operations per second |   Relative (latency) |
|------------|---------------------------------|-------------------------|----------------------|
| orjson     |                            0.07 |                 13244.4 |                 1    |
| ujson      |                            0.2  |                  4869   |                 2.73 |
| rapidjson  |                            0.27 |                  3768.1 |                 3.54 |
| simplejson |                            0.44 |                  2290.3 |                 5.83 |
| json       |                            0.35 |                  2821.8 |                 4.73 |

#### github.json deserialization

| Library    |   Median latency (milliseconds) |   Operations per second |   Relative (latency) |
|------------|---------------------------------|-------------------------|----------------------|
| orjson     |                            0.21 |                  4811.7 |                 1    |
| ujson      |                            0.28 |                  3540.1 |                 1.36 |
| rapidjson  |                            0.31 |                  3207.8 |                 1.5  |
| simplejson |                            0.29 |                  3443.9 |                 1.4  |
| json       |                            0.33 |                  3046.7 |                 1.59 |

#### citm_catalog.json serialization

| Library    |   Median latency (milliseconds) |   Operations per second |   Relative (latency) |
|------------|---------------------------------|-------------------------|----------------------|
| orjson     |                            1.02 |                   978.7 |                 1    |
| ujson      |                            3.48 |                   287.7 |                 3.4  |
| rapidjson  |                            3.48 |                   287.3 |                 3.4  |
| simplejson |                           10.85 |                    91.3 |                10.62 |
| json       |                            6.79 |                   147.4 |                 6.64 |

#### citm_catalog.json deserialization

| Library    |   Median latency (milliseconds) |   Operations per second |   Relative (latency) |
|------------|---------------------------------|-------------------------|----------------------|
| orjson     |                            4.38 |                   228.3 |                 1    |
| ujson      |                            5.58 |                   179.2 |                 1.27 |
| rapidjson  |                            7.52 |                   132.2 |                 1.72 |
| simplejson |                            7.4  |                   135.2 |                 1.69 |
| json       |                            7.88 |                   126.9 |                 1.8  |

#### canada.json serialization

| Library    |   Median latency (milliseconds) |   Operations per second |   Relative (latency) |
|------------|---------------------------------|-------------------------|----------------------|
| orjson     |                            4.69 |                   209.7 |                 1    |
| ujson      |                           20.03 |                    49.9 |                 4.27 |
| rapidjson  |                           61.8  |                    16.2 |                13.17 |
| simplejson |                           79.63 |                    12.6 |                16.96 |
| json       |                           62.73 |                    15.9 |                13.36 |

#### canada.json deserialization

| Library    |   Median latency (milliseconds) |   Operations per second |   Relative (latency) |
|------------|---------------------------------|-------------------------|----------------------|
| orjson     |                           10.14 |                    98.5 |                 1    |
| ujson      |                           15.58 |                    64.4 |                 1.54 |
| rapidjson  |                           37.3  |                    26.8 |                 3.68 |
| simplejson |                           36.4  |                    27.5 |                 3.59 |
| json       |                           37.24 |                    27.2 |                 3.67 |

If a row is blank, the library did not serialize and deserialize the fixture without
modifying it, e.g., returning different values for floating point numbers.

### Memory

orjson's memory usage when deserializing is similar to or lower than
the standard library and other third-party libraries.

This measures, in the first column, RSS after importing a library and reading
the fixture, and in the second column, increases in RSS after repeatedly
calling `loads()` on the fixture.

#### twitter.json

| Library    |   import, read() RSS (MiB) |   loads() increase in RSS (MiB) |
|------------|----------------------------|---------------------------------|
| orjson     |                       13.6 |                             2.5 |
| ujson      |                       14.1 |                             4.1 |
| rapidjson  |                       14.7 |                             6.4 |
| simplejson |                       13.5 |                             2.5 |
| json       |                       13   |                             2.3 |

#### github.json

| Library    |   import, read() RSS (MiB) |   loads() increase in RSS (MiB) |
|------------|----------------------------|---------------------------------|
| orjson     |                       13.1 |                             0.3 |
| ujson      |                       13.7 |                             0.3 |
| rapidjson  |                       14   |                             0.7 |
| simplejson |                       12.7 |                             0.3 |
| json       |                       12.3 |                             0.1 |

#### citm_catalog.json

| Library    |   import, read() RSS (MiB) |   loads() increase in RSS (MiB) |
|------------|----------------------------|---------------------------------|
| orjson     |                       15   |                             7.7 |
| ujson      |                       15.4 |                            11.1 |
| rapidjson  |                       15.8 |                            29.4 |
| simplejson |                       14.5 |                            30.7 |
| json       |                       14.2 |                            27.2 |

#### canada.json

| Library    |   import, read() RSS (MiB) |   loads() increase in RSS (MiB) |
|------------|----------------------------|---------------------------------|
| orjson     |                       17.3 |                            15.7 |
| ujson      |                       17.7 |                            17.4 |
| rapidjson  |                       18.3 |                            17.9 |
| simplejson |                       17.1 |                            19.6 |
| json       |                       16.8 |                            19.4 |

### Reproducing

The above was measured using Python 3.8.2 on Linux (x86_64) with
orjson 2.6.4, ujson 2.0.3, python-rapidson 0.9.1, and simplejson 3.17.0.

The latency results can be reproduced using the `pybench` and `graph`
scripts. The memory results can be reproduced using the `pymem` script.

## Packaging

To package orjson requires [Rust](https://www.rust-lang.org/) on the
 nightly channel and the [maturin](https://github.com/PyO3/maturin)
build tool. maturin can be installed from PyPI or packaged as
well. maturin can be invoked like:

```sh
maturin build --no-sdist --release --strip --manylinux off
```

orjson is tested for amd64 and aarch64 on Linux, macOS, and Windows. It
may not work on 32-bit targets. It should be compiled with
`-C target-feature=+sse2` on amd64 and `-C target-feature=+neon` on arm7. musl
libc is not supported, but building with `-C target-feature=-crt-static`
will probably work.

There are no runtime dependencies other than libc.

orjson's tests are included in the source distribution on PyPI. It is
necessarily to install dependencies from PyPI specified in
`test/requirements.txt`. These require a C compiler. The tests do not
make network requests.

The tests should be run as part of the build.

## License

orjson was written by ijl <<ijl@mailbox.org>>, copyright 2018 - 2020, licensed
under either the Apache 2 or MIT licenses.
