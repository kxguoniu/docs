[TOC]
# asyncio 之 format_helpers.py
## 摘要
## def _get_function_source
返回函数所在的文件名，以及在文件中的位置
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
`funcname(arg1, arg2[, ...]) at filename:line`
```python
def _format_callback_source(func, args):
    func_repr = _format_callback(func, args, None)
    source = _get_function_source(func)
    if source:
        func_repr += f' at {source[0]}:{source[1]}'
    return func_repr
```
## def _format_args_and_kwargs
`(arg1, arg2[, ...], kwarg=arg, kwarg2=arg[, ...])`
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
`funcname(arg1, arg2[, ...], kwarg=arg, kwarg2=arg[, ...])`
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