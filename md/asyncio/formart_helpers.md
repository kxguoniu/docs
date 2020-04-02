[TOC]
## 摘要
`format_helpers`打印log的帮助函数，主要作用是寻找方法所在的位置或者输出方法执行时的语句。通过日志可以了解到程序执行了那个方法，方法所在的位置以及执行方法时传入的参数等。
## def _get_function_source
返回函数所在的文件名，以及在文件中的行号
```python
def _get_function_source(func):
    func = inspect.unwrap(func)
    if inspect.isfunction(func):
        code = func.__code__
        return (code.co_filename, code.co_firstlineno)
    if isinstance(func, functools.partial):
        return _get_function_source(func.func)
    if isinstance(func, functools.partialmethod):
        return _get_function_source(func.func)
    return None
```
## def _format_callback_source
返回函数调用的参数以及在文件中的位置，只有位置参数
返回内容格式1 `funcname(arg1, arg2[, ...]) at filename:line`
返回内容格式2 `funcname(arg1, ..., kwarg1=arg1, ...)(arg1, ...) at filename:line`
```python
def _format_callback_source(func, args):
    func_repr = _format_callback(func, args, None)
    source = _get_function_source(func)
    if source:
        func_repr += f' at {source[0]}:{source[1]}'
    return func_repr
```
## def _format_args_and_kwargs
返回内容格式 `(arg1, arg2[, ...], kwarg=arg, kwarg2=arg[, ...])`
```python
def _format_args_and_kwargs(args, kwargs):
    # 使用reprlib限制输出的长度
    items = []
    if args:
        items.extend(reprlib.repr(arg) for arg in args)
    if kwargs:
        items.extend(f'{k}={reprlib.repr(v)}' for k, v in kwargs.items())
    return '({})'.format(', '.join(items))
```
## def _format_callback
返回函数调用的参数，位置参数和字典参数
返回内容格式1 `funcname(arg1, arg2[, ...], kwarg=arg, kwarg2=arg[, ...])`
返回内容格式2 `funcname(arg1, ..., kwarg1, ...)(arg1, ..., kwarg1, ...)`
```python
def _format_callback(func, args, kwargs, suffix=''):
    if isinstance(func, functools.partial):
        suffix = _format_args_and_kwargs(args, kwargs) + suffix
        return _format_callback(func.func, func.args, func.keywords, suffix)

    if hasattr(func, '__qualname__') and func.__qualname__:
        func_repr = func.__qualname__
    elif hasattr(func, '__name__') and func.__name__:
        func_repr = func.__name__
    else:
        func_repr = repr(func)

    func_repr += _format_args_and_kwargs(args, kwargs)
    if suffix:
        func_repr += suffix
    return func_repr
```
## def extract_stack
代码回溯限制
```python
def extract_stack(f=None, limit=None):
    """Replacement for traceback.extract_stack() that only does the
    necessary work for asyncio debug mode.
    """
    if f is None:
        f = sys._getframe().f_back
    if limit is None:
        # Limit the amount of work to a reasonable amount, as extract_stack()
        # can be called for each coroutine and future in debug mode.
        limit = constants.DEBUG_STACK_DEPTH
    stack = traceback.StackSummary.extract(traceback.walk_stack(f),
                                           limit=limit,
                                           lookup_lines=False)
    stack.reverse()
    return stack
```