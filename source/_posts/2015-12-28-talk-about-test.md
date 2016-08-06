title: 浅谈测试
date: 2015-12-28 15:32:00
tags:
- Python
- 测试
categories:
- Develop
---

![You just keep on trying till you run out of cake.](/media/2015/12/portal.png)

这篇文章是写了一段时间测试之后的简单总结，主要讨论在 Python 语言环境下的功能测试和单元测试。

<!-- more -->

### 0. 为什么需要测试？

测试是验证我们编写的软件的正确性、完整性和安全性的过程，目的就是验证我们编写的程序能够在各种情况下能表现出我们希望的结果。

一旦项目变得庞大之后，如果拥有合理的测试，并且能与 CI 结合起来，能够很大地提高项目的可维护性。一定量的测试可以帮助我们发现逻辑错误或者 API Break 等错误，在一定程度上还可以帮助我们重构代码，让我们发现过分庞大的程序模块。

尽管包括我在内的很多开发人员都不是很喜欢写测试，但是本着对项目负责的原则，测试确实是必不可少的。

### 1. 功能测试还是单元测试？

大概日常开发过程中，作为一个开发人员而非测试人员，可能能接触到最多的测试就是功能测试和单元测试了吧。

在写项目的测试之前，我们应该决定好对哪些模块应该做单元测试，对哪些模块应该做功能测试。

单元测试是对软件中最小的可测试部件，也就是软件模块进行正确性验证的过程。

简单一点说，就是针对软件中单一的类或者单一的函数或者方法的测试。既然只针对单一模块的测试，那么在本次测试的过程之中，不应该受其他模块的干扰，举个例子说，模块 A 调用了模块 B 和模块 C，我们需要去对模块 A 进行单元测试，那么我们测试模块 A 的过程中不应该再真实得调用 B 或者 C 模块，或者真实地从 B 或者 C 模块获取数据，因为我们这次单元测试是仅仅针对模块 A 的。那么如何避免 B 和 C 模块的干扰？最常见的办法就是把模块 A 中的 B 模块和 C 模块替换掉，让模块 B 和 C 直接按照我们要求的方式响应，这就是测试中常见的模拟对象 mock，关于 mock 会在下一小节之中稍详细地提及。

功能测试顾名思义就是对程序功能的测试，也就是对模块或者软件功能方面上的正确性验证。

尽管单元测试在大部分的情况下能够满足我们对某些测试的需要，但是有些模块使用单元测试不适合使用单元测试，或者并不太好写单元测试。前者的例子是 Model 最底层中对数据库的操作的测试，因为在最底层已经基本是直接调用数据库 Binding 过来相关的 API 了，这里与其用单元测试去测试 API 调用，不如直接测数据库操作是否成功落到数据库上更实际一些；而后者的例子是 Tornado 作为 Web 框架使用时对 Hanlder 的测试，我们可能针对 Handler 做单元测试很麻烦，而 Tornado 本身提供的 `tornado.testing`，基本就属于功能测试的范畴 。当然功能测试有时候也是需要 mock 掉其他的模块，比如在测试 Tornado 的某个 Handler 的时候，可以把这个 Handler 的外部调用都 mock 掉，使测试的范围尽量小一些。

### 2. Mock 和 Fixture

Mock 是对真实调用或者真实对象的模拟，以方便我们确定测试中外部调用的行为，从而减小我们测试的针对范围，也方便了分模块开发下软件模块的测试。

简单举一个例子，在 `cal.py` 中有这样一个获取一个随机数然后乘以 2 的函数：

```python
# !/bin/env python2
# cal.py

from radom import randint


def double_rand():
    return randint(1, 20) * 2
```

如果我们希望测试它，但是 `randint` 的返回值是不确定的，那么我们就应该通过将 `randint` 这个函数 mock 掉，让它返回一个确定的结果，从而方面我们的测试，以下利用 Python 2.7 中标准库 `unittest` 和第三方库 `mock` 来测试：

```python
# !/bin/env python2
# test_cal.py

from unittest import TestCase

import mock

from cal import double_rand


class TestCal(TestCase):

    @mock.patch("cal.randint")
    def test_double_rand(self, _randint):
        _randint.return_value = 2
        self.assertEqual(double_rand(), 4)
```

这里我们将 `cal` 模块中 `randint` mock 掉了，并确定其返回值为 2，最后只测试了 `double_rand` 的本身逻辑，也就是乘 2。

Fixture 是我们进行测试的一个可重用的基础环境，简单点说，就是一个设置好的已知的测试环境，在进行完单个测试用例或者进行完整个测试之后，可以恢复到原始的状态。

比如说我们测试 Model 层是否能成功落在数据库之中，如果测试插入，我们就需要在测试前有一个干净的数据库，在测试完成之后又将其恢复到干净的状态，如果测试查询，那么我们还需要对数据库进行数据的提前插入，保证是一个已知的可查询状态，在测试完毕后也是需要回复到干净的状态。

一般来说 Fixture 都通过一个 `setUp` 函数来设置初始的状态，在测试结束后通过 `tearDown` 函数来恢复原始的状态。

这里以 Python 2.7 中的标准库 `unittest` 和第三方库 `pytest` 为例子，写两个 Fixture：

```python
# !/bin/env python2


class TestBalaModel(TestCase):

    def setUp(self):
        self.connection  = engine.connet()
        self.transaction = connection.begin()
        self.session = scoped_session(db_session)

    def test_insert(self):
        pass

    def tearDown(self):
        self.session.remove()
        self.transaction.rollback()
        self.connection.close()


@pytest.fixture(scope="function")
def session(request):
    connection  = engine.connet()
    transaction = connection.begin()
    session = scoped_session(db_session)

    def tearDown():
        session.remove()
        transaction.rollback()
        connection.close()

    request.addfinalizer(tearDown)
    return session
```

前者是 `unittest` 风格的 Fixture，在测试用例中所有的测试运行之前先运行 `self.setUp` 来生成一个基础环境，在所有的测试运行完毕后通过运行 `self.tearDown` 来恢复之前的环境。后者是 `pytest` 风格的 Fixture，通过一个装饰器来定义，并定义了其作用范围为函数内，在一个测试函数中引入这个 session，会提供我们设定好的 session 变量，在离开测试函数后会通过 `tearDown` 来回复之前的环境。

### 3. Python 测试相关包的选择

虽然 `The Zen of Python` 有说 "There should be one-- and preferably only one --obvious way to do it."，但是讲道理来说测试相关的包早就不止一个了。

Python 2/3 的提供的官方包 `unittest` 提供了一套可行的测试套件，包括一些测试用的 utils 和 Fixture，基本都是以测试用例的实例的方法形式使用的，用起来没有障碍，但是感觉自动调用的 Fixture 作用域最低也是在 class 级别，作用域比较大，并没有提供 funtion 级别的 Fixture。

Python 2/3 还提供了 `doctest` 这个非常有趣的库，可以提取 docstring 中的内容，作为测试用例使用，印象中 Rustlang 和其他语言也有这种测试的支持。

Python2 的第三方包以及 Python 3 的标准包 `mock` 提供了用于 mock 的工具以及相关测试用的 utils，比如被 mock 对象的调用次数一类。

`Pytest` 和 `Nose` 也提供了一套与 `unittest` 平行的测试套件，其测试方式更加灵活，可以在一个函数内完成一个测试用例，可以用插件拓展，也提供的 function 级别的 Fixture，同样兼容标准库之中的 `unittest`。其中 `Pytest` 的交互更加友好一些，提供错误或者异常时各变量的值，但是 `Pytest` 的文档读起来与其说是文档，更像是教程一些。

### 4. 一些实践

#### a. 参数化的测试

在测试之中，可能会遇到类似这样的逻辑流：

```python
# !/bin/env python2


def foo(a, b):
    if a == 1:
        if b == 1:
            pass
        else:
            pass
    else:
        if b == 1:
            pass
        else:
            pass
```

对于这种情况，我们一般来说需要测试 4 次，也就是 a 是 1 以及 b 是 1 的情况，a 是 1 以及 b 不是 1 的情况，a 不是 1 以及 b 是 1 的情况，a 不是 1 以及 b 是 1 的情况，这样如果我们单纯使用传统的 `unittest`，可能写四段重复度较高的测试。这里可以使用 `Pytest` 提供的装饰器 `pytest.mark.parametrize`，提供参数化的输入参数，并且在运行测试的时候会拆分成几个测试运行，详见 [Parametrizing fixtures and test functions](http://pytest.org/latest/parametrize.html#parametrized-test-functions)。如果使用的是 `nose`，可以考虑用 [nose-parameterized](https://github.com/wolever/nose-parameterized) 实现差不多的功能。

#### b. mock 构造函数

不建议使用 mock 库去完成这一项功能，如果非要 mock 掉构造函数，可以使用继承的方法。

```python
# !/bin/env python2


class SomeThing(object):

    def __init__(a, b):
        self.a = a
        self.b = self.foo(b)


def test_some_thing():

    class MockedSomeThing(SomeThing):
    
        def __init__():
            # Fill in mock here

    _something = MockedSomeThing()
```

#### c. mock 文件 IO 相关

可以参考 `mock` 提供的例子 [Mocking the builtin open used as a context manager](http://www.voidspace.org.uk/python/mock/compare.html#mocking-the-builtin-open-used-as-a-context-manager)，mock 掉 `open` 之后使用 `StringIO` 来代替原始的文件 IO 流，这种方式不仅可以支持直接 `open` 返回可读的对象，同样可以支持上下文管理器。

```python
# !/bin/env python2


def test_file():
    with mock.patch("__builtin__.open") as my_mock:
        my_mock.return_value = io.StringIO("Fake IO")
```

#### d. 测试 Tornado 的需要认证的 Handler

`Tornado` 提供了 `tornado.web.authenticated` 这个装饰器来方便对 Handler 处理的请求做认证，当我们做测试的时候，如果需要模拟认证过的用户，可以通过 mock 掉 `get_secure_cookie` 或者 `get_current_user` 来实现，毕竟这个 `tornado.web.authenticated` 这个装饰器也是通过前两者实现的。

```python
# !/bin/env python2


class TestAPI(AsyncHTTPTestCase):

    def get_app(self):
        return Application([("/", BalaHandler)], cookie_secret="badsecret")

    def test_req(self):
        with mock.patch.object(BalaHandler, "get_current_user") as _get_current_user:
            _get_current_user.return_value = "admin"
            # Fill in test code
```

#### e. 测试 Logging 相关

`textfixtures` 这个第三方包提供了一套方便的工具让我们测试 `Logging` 相关，测试 logger 主要是测试两个方面，一是测试 logger 的各种设置与自己期望是否一致：

```python
# !/bin/env python2

import logging
from unittest import TestCase

from testfixtures import Comparison, compare

from bala import logger


class TestLogger(TestCase):

    def test_config(self):
        compare(logger.level, 20)
        compare([Comparison("logging.StreamHandler",
                            stream=sys.stderr,
                            formatter=Comparison("logging.Formatter",
                            _fmt="%(levelname)s %(message)s",
                            strict=False),
                            level=logging.NOSET),
                            strict=False)],
                logger.handlers)
```

二是测试信息是否正常被记录：

```python
# !/bin/env python2

import logging

from testfixtures imoport LogCapture


def test():
    with LogCapture() as l:
        logger = logging.getLogger()
        logger.info("a message")
        logger.error("an error")
        l.check(
            ("root", "INFO", "a message"),
            ("root", "ERROR", "an error"),
        )
```

------

参考资料：

1. http://alexmic.net/flask-sqlalchemy-pytest/
2. http://pytest.org/latest/parametrize.html#parametrized-test-functions
3. https://github.com/wolever/nose-parameterized
4. http://stackoverflow.com/questions/17836939/mocking-init-for-unittesting
5. http://stackoverflow.com/questions/8746586/getting-an-actual-return-value-for-a-mocked-file-read
6. http://www.voidspace.org.uk/python/mock/compare.html#mocking-the-builtin-open-used-as-a-context-manager 
7. http://stackoverflow.com/questions/18285947/how-to-use-a-test-tornado-server-handler-that-authenticates-a-user-via-a-secure
8. https://pythonhosted.org/testfixtures/logging.html
