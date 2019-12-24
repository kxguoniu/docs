[TOC]
# asyncio 之 runners.py
## 摘要
## def run
```python
def run(main, *, debug=False):
    """
        async def main():
            await asyncio.sleep(1)
            print('hello')

        asyncio.run(main())
    """
	# 如果线程已经存在正在运行的事件循环，抛出异常
    if events._get_running_loop() is not None:
        raise RuntimeError(
            "asyncio.run() cannot be called from a running event loop")
	# 运行的函数必须是一个协程
    if not coroutines.iscoroutine(main):
        raise ValueError("a coroutine was expected, got {!r}".format(main))
	# 创建一个新的事件循环
    loop = events.new_event_loop()
	# 设置事件循环，设置调试参数，运行函数
    try:
        events.set_event_loop(loop)
        loop.set_debug(debug)
        return loop.run_until_complete(main)
    finally:
		# 执行完毕之后，取消事件循环中所有的任务，关闭异步生成器，关闭事件循环
        try:
            _cancel_all_tasks(loop)
            loop.run_until_complete(loop.shutdown_asyncgens())
        finally:
            events.set_event_loop(None)
            loop.close()
```
## def _cancel_all_tasks
```python
def _cancel_all_tasks(loop):
	# 取出事件循环中的所有未完成的 task
    to_cancel = tasks.all_tasks(loop)
    if not to_cancel:
        return
	# 取消所有的 task
    for task in to_cancel:
        task.cancel()
	# 运行事件循环，把所有取消的任务执行完毕。
    loop.run_until_complete(
        tasks.gather(*to_cancel, loop=loop, return_exceptions=True))
	# 循环需要取消的任务，如果有异常则记录下来
    for task in to_cancel:
        if task.cancelled():
            continue
        if task.exception() is not None:
            loop.call_exception_handler({
                'message': 'unhandled exception during asyncio.run() shutdown',
                'exception': task.exception(),
                'task': task,
            })

```