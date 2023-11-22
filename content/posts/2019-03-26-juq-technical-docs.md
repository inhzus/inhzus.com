---
title: Juq Technical Documentation
date: 2019-03-26 14:17:42
tags: [Python, Tech, Juq]
description: Some exellent technical practice in the development of juq library
---

Some exellent technical practice in the development of [Juq](https://github.com/inhzus/juq).

### Decorators for functions in a module

```python
def decarate_module(module, decorator: types.FunctionType):
    for name in dir(module):
        method = getattr(module, name)
        if isinstance(method, types.FunctionType):
            setattr(module, name, decorator(method))
    return module
```

Usage: e.g. Serialize all return values for functions in a module

### dataclass

#### Reason

e.g.

```python
def Example(object):
    def __init__(self, a: int, b: str, c: str, d: str, e: str, f: str, g: str, ...):
        self.a = a
        self.b = b
        self.c = c
        ...
```

Common method of constructor is a waste of time and code space.

<!--more-->

#### Then I found

```python
from collections import namedtuple

Example = namedtuple('Example', 'a b c d e f g')
```

However, namedtuple is not friendly to type hints. The [method](<https://stackoverflow.com/questions/34269772/type-hints-in-namedtuple/34269877>) is complex and not elegent.

#### Finally

```python
from dataclasses import dataclass

# @dataclass(repr=False)
@dataclass
class Example:
    a: int
    b: str
    c: str
    d: str
    e: str
    f: str
    g: str
```

Dataclass is excellent and supplies some basic function.

### Filter function's useless params

When using a series of functions with the same return value and similar parameters, it's convenient to use a map to direct to different function on different demand. Ignorarable keyword arguments are useful here.

e.g.

```python
>>> def apple(a, b, **_):
...     return '{}{}'.format(a, b)
...
>>> def blue(a, **_):
...     return a
...
>>> input_dict = {'func': 'blue', 'a': 'q', 'b': 'e'}
>>> globals()[input_dict['func']](**input_dict)
q
>>> input_dict['func'] = 'apple'
>>> globals()[input_dict['func']](**input_dict)
qe
```

### Command line parser

Python supplies a module - [argparse](https://docs.python.org/3/library/argparse.html) - which makes it easy to write user-friendly command-line interfaces.

In juq, I need to parse multiple nested sub-commands. Set sub-parser in a seprerated function is the best pratice imo. The example is as following:

```python
import argparse


def set_user_parser(user_parser: argparse.ArgumentParser):
    _parser = user_parser.add_subparsers(dest='user', help='identifier: login/id')
    _parser.required = True

    info = _parser.add_parser('info', help='get user detailed info')
    info.add_argument('id_', help='user id, self by default', nargs='?',
                      default='', type=str, metavar='id')


def set_doc_parser(doc_parser: argparse.ArgumentParser):
    pass
    

parser = argparse.ArgumentParser()
sub_parser = parser.add_subparsers(dest='sub')
sub_parser.required = True

set_user_parser(sub_parser.add_parser('user', help='user help'))
set_doc_parser(sub_parser.add_parser('doc'))

parser.parse_args()
```

As the example above, some info should be noticed.

#### Subparser

To add sub parsers, it is a process which is similar to create a list: create an empty list firstly and fill in the list then.

```python
parser = argparse.ArgumentParser()
sub_parsers = parser.add_subparsers()
first_sub_parser = sub_parsers.add_parser('first')
```

It seems that creating an empty subparser list is useless, however, it's easy to configure parser with these steps.

#### Which subparser is used

`dest` parameter is used for this case. As the code showed above, specify `dest` parameter and we can get which subparser is used.

```python
>>> parser = argparse.ArgumentParser()
>>> sub_parser = parser.add_subparsers(dest='sub')
>>> sub_parser.add_parser('user')
>>> parser.parse_args(['user'])`
Namespace(sub='user')
```

#### Argument without value

To set an argument without value, you can use:

```python
>>> parser.add_argument('--public', '-p', action='store_const', const=1, default=0)
>>> parser.parse_args(['-p'])
Namespace(public=1)
>>> parser.parse_args([])
Namespace(public=0)
```

`action` parameter can be specified as: store_const, store_true, store_false. With specifying `const` parameter and `default` parameter, `action` parameter can be used as you want.