# 解决 ROS1 在 Fedora 38 RISC-V + Python 3.11 下的死循环问题
在 Fedora 38 RISC-V 平台下源码安装了 ROS1 之后，在尝试启动 `roscore` 时发现出现了死循环现象，进程吃满整个单个CPU，遂展开调查

## 过程
在死循环期间连按 Ctrl-C 可以发现此时进程处于 `RospyLogger` 的 `findLogger()` 方法内，遂定位问题理应出自此函数

```python
    def findCaller(self, *args, **kwargs):
        """
        Find the stack frame of the caller so that we can note the source
        file name, line number, and function name with class name if possible.
        """
        file_name, lineno, func_name = super(RospyLogger, self).findCaller(*args, **kwargs)[:3]
        file_name = os.path.normcase(file_name)

        f = inspect.currentframe()
        if f is not None:
            f = f.f_back
        while hasattr(f, "f_code"):
            # Search for the right frame using the data already found by parent class.
            co = f.f_code
            filename = os.path.normcase(co.co_filename)
            if filename == file_name and f.f_lineno == lineno and co.co_name == func_name:
                break
            if f.f_back:
                f = f.f_back

        # Jump up two more frames, as the logger methods have been double wrapped.
        if f is not None and f.f_back and f.f_code and f.f_code.co_name == '_base_logger':
            f = f.f_back
            if f.f_back:
                f = f.f_back
        co = f.f_code
        func_name = co.co_name

        # Now extend the function name with class name, if available.
        try:
            class_name = f.f_locals['self'].__class__.__name__
            func_name = '%s.%s' % (class_name, func_name)
        except KeyError:  # if the function is unbound, there is no self.
            pass

        if sys.version_info > (3, 2):
            # Dummy last argument to match Python3 return type
            return co.co_filename, f.f_lineno, func_name, None
        else:
            return co.co_filename, f.f_lineno, func_name
```

该函数位于 roslogging.py(https://github.com/ros/ros_comm/blob/030e132884d613e49a576d4339f0b8ec6f75d2d8/tools/rosgraph/src/rosgraph/roslogging.py#L53) 内，`RospyLogger` 继承了 `logging` 类并重写了其 `findLogger()` 成员方法

从代码行为可以很容易理解逻辑，先调用父类 `logging` 的 `findLogger()` 函数获得调用 `log()` 和 `info()` 的 caller 所在的文件名、行数和函数名，随后再自行调用 inspect.currentFrame() 获取当前函数调用栈，并依次向上遍历，直到找到对应的栈帧对象后跳出循环（笔者认为这个行为相当扭曲，因为这就导致实际上总共执行了两次栈帧遍历），并重新定义了类名和函数名随后返回

一开始以为是架构导致的问题，为了对比，在 x86 平台上安装了 ROS1，并同时修改 RISC-V 端和 x86 端的代码，使其输出的第一行为 `super(RospyLogger, self).findCaller(*args, **kwargs)[:3]` 返回的文件名、行数和函数名，并在从下往上遍历栈帧时依次输出"->"和当前栈对象。

x86 平台上的输出符合逻辑，在遍历4次后找到了 parent.py 的栈对象
```
/opt/ros/noetic/lib/python3/dist-packages/roslaunch/parent.py, 301, start
-> <frame at 0x1097fb0, file '/opt/ros/noetic/lib/python3/dist-packages/rosgraph/roslogging.py', line 62, code findCaller>
-> <frame at 0x10cb000, file '/usr/lib/python3.8/logging/__init__.py', line 1577, code _log>,/usr/lib/python3.8/logging/__init__.py,1577,_log
-> <frame at 0x7f3ab6416580, file '/usr/lib/python3.8/logging/__init__.py', line 1446, code info>,/usr/lib/python3.8/logging/__init__.py,1446,info
-> <frame at 0x7f3ab6341040, file '/opt/ros/noetic/lib/python3/dist-packages/roslaunch/parent.py', line 301, code start>,/opt/ros/noetic/lib/python3/dist-packages/roslaunch/parent.py,301,start
```

而 RISC-V 平台上的输出则出现死循环
```
/home/milkice/ros_catkin_ws/install_isolated/lib/python3.11/site-packages/rosgraph/roslogging.py,58,findCaller
-> <frame at 0xffffff6c67a180, file '/home/milkice/ros_catkin_ws/install_isolated/lib/python3.11/site-packages/rosgraph/roslogging.py', line 62, code findCaller>
-> <frame at 0xffffff6c688d60, file '/usr/lib64/python3.11/logging/__init__.py', line 1622, code _log>,/usr/lib64/python3.11/logging/__init__.py,1622,_log
-> <frame at 0xffffff6c66fc60, file '/usr/lib64/python3.11/logging/__init__.py', line 1489, code info>,/usr/lib64/python3.11/logging/__init__.py,1489,info
-> <frame at 0xffffff6c699540, file '/home/milkice/ros_catkin_ws/install_isolated/lib/python3.11/site-packages/roslaunch/parent.py', line 301, code start>,/home/milkice/ros_catkin_ws/install_isolated/lib/python3.11/site-packages/roslaunch/parent.py,301,start
-> <frame at 0xffffff6c6aaf20, file '/home/milkice/ros_catkin_ws/install_isolated/lib/python3.11/site-packages/roslaunch/__init__.py', line 347, code main>,/home/milkice/ros_catkin_ws/install_isolated/lib/python3.11/site-packages/roslaunch/__init__.py,347,main
-> <frame at 0xffffff6c699600, file '/home/milkice/ros_catkin_ws/install_isolated/bin/roscore', line 84, code <module>>,/home/milkice/ros_catkin_ws/install_isolated/bin/roscore,84,<module>
-> <frame at 0xffffff6c699600, file '/home/milkice/ros_catkin_ws/install_isolated/bin/roscore', line 84, code <module>>,/home/milkice/ros_catkin_ws/install_isolated/bin/roscore,84,<module>
-> <frame at 0xffffff6c699600, file '/home/milkice/ros_catkin_ws/install_isolated/bin/roscore', line 84, code <module>>,/home/milkice/ros_catkin_ws/install_isolated/bin/roscore,84,<module>
-> <frame at 0xffffff6c699600, file '/home/milkice/ros_catkin_ws/install_isolated/bin/roscore', line 84, code <module>>,/home/milkice/ros_catkin_ws/install_isolated/bin/roscore,84,<module>
-> <frame at 0xffffff6c699600, file '/home/milkice/ros_catkin_ws/install_isolated/bin/roscore', line 84, code <module>>,/home/milkice/ros_catkin_ws/install_isolated/bin/roscore,84,<module>
-> <frame at 0xffffff6c699600, file '/home/milkice/ros_catkin_ws/install_isolated/bin/roscore', line 84, code <module>>,/home/milkice/ros_catkin_ws/install_isolated/bin/roscore,84,<module>
...
```

其实此时 /home/milkice/ros_catkin_ws/install_isolated/bin/roscore:84 已经是栈顶，而导致死循环的关键原因便是[此处](https://github.com/ros/ros_comm/blob/030e132884d613e49a576d4339f0b8ec6f75d2d8/tools/rosgraph/src/rosgraph/roslogging.py#L70)
```python
            if f.f_back:
                f = f.f_back
```
由于栈顶 frame 对象的 f_back 属性为 None，而该循环仅仅只是当 f_back 不为 None 时将 f 指向 f_back，并没有对 None 的情况跳出循环，导致每次循环都是该栈顶frame，以至于出现死循环现象
同时也能观察到 x86 和 RISC-V 平台对 `super(RospyLogger, self).findCaller(*args, **kwargs)[:3]` 返回的文件名、行数和函数名存在不一的情况

后来查阅 Python 3.11 logging 模块变更日志发现 [此commit](https://github.com/python/cpython/pull/28287/files) 引入了上述问题，`logging` 本身的 `findCaller()` 方法也是用于寻找在调用 `log()` 和 `info()` 等日志函数时的调用者，其本身也会沿着堆栈向上查找最先不属于 `logging` 类的栈帧，但该 commit 修改了遍历逻辑，引入了 `_is_internal_frame()` 函数以判断当前栈帧是否为 `logging` 内部所有，而忽略了调用栈里可能有 `logging` 子类的情况

这就导致，原本的逻辑下，`logging` 本身的 `findCaller()` 寻找到自身和 `RospyLogger` 的方法都会跳过，最后返回 `parent.py` 这个外部函数所在的文件，但新逻辑下，在跳过 `logging.py` 的 `findCaller()` 后，下一个 `roslogging.py` 的 `findCaller()` 已被判定为外部函数而返回，进而导致上述死循环的产生

下面这张图中的链表表示调用 `super(RospyLogger, self).findCaller(*args, **kwargs)[:3]` 后，`findCaller()`的调用栈，显示了 Python 3.8 和 Python 3.11 下 `logging` 的 `findCaller()` 不同的行为，返回的结果已用绿色标注

![logging](https://github.com/fedora-riscv/roslogging-bug-docs/assets/5274559/b651c3ec-199d-4bd8-a166-4ecfd4a2c356)

## 修复
ROS 原本的逻辑极具依赖 Python 自身 `logging` 对 `findCaller()` 的实现，这里重写了 `RospyLogger` 下的 `findCaller()` 函数，不再依赖父类的 `findCaller()`
```python
    def findCaller(self, *args, **kwargs):
        """
        Find the stack frame of the caller so that we can note the source
        file name, line number, and function name with class name if possible.
        """

        f = inspect.currentframe()
        while f.f_back:
            f = f.f_back
            # Get locals from f_locals
            f_class = f.f_locals.get('self', None)
            if not f_class:
                # Just continue first
                continue 
            if f_class.__class__ is not RospyLogger:
                # Skip RospyLogger
                break
        co = f.f_code
        func_name = co.co_name

        # Now extend the function name with class name, if available.
        try:
            class_name = f.f_locals['self'].__class__.__name__
            func_name = '%s.%s' % (class_name, func_name)
        except KeyError:  # if the function is unbound, there is no self.
            pass

        if sys.version_info > (3, 2):
            # Dummy last argument to match Python3 return type
            return co.co_filename, f.f_lineno, func_name, None
        else:
            return co.co_filename, f.f_lineno, func_name
```

经过测试，上述代码可在 Python 3.8 和 Python 3.11 上测试通过，如果没有问题后续将发起pr

感谢 [U2FsdGVkX1](https://github.com/U2FsdGVkX1) 在解决本问题时提供的帮助
