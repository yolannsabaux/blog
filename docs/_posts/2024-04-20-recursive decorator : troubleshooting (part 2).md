---
title: "Recursive Decorator: An AST Approach (Part 2)"
permalink: /recursive-decorator/2/
layout: post
---

* * *

I initially thought to write it in one part, but it seemed too intense to do so. I divided it into three parts:
- [Part 1](/blog/recursive-decorator/1/): provides a quick description of what we aim to achieve, along with a clunky draft of a recursive decorator.
- [Part 2](/blog/recursive-decorator/2/): corrects the draft to have a functional recursive decorator.
- [Part 3](/blog/recursive-decorator/3/): utilizes the full set of tools from `ast` to make it even more beautiful.

* * *

## Troubleshooting: Why Isn't My Decorator Working?

**Indeed...**
If you applied the decorator to the following snippet, it likely didn't work as expected:
```python
def bar():
    print("I'm bar")

@decorator
def foo():
    print("I'm foo")
    bar()
```
But don't worry, there are just a few adjustments we still need to apply, and then it will work just fine!

### Removing the `recursive_decorator` from the `decorator_list`
If we leave the recursive decorator in the list of our main decorated function, it will call the decorator again, failing at the `inspect.py` step.
The reason is: "when the function is called, the decorator runs again - so, it gets some re-entrancy in inspect.getsource, at which point it fails." [stackoverflow](https://stackoverflow.com/questions/75696056/pythons-inspect-getsource-throws-error-if-used-in-a-decorator)

There are two solutions:
1) Don't decorate `foo` with `@decorator`, but instead assign `foo = decorator(foo)` before calling `foo`.
2) During the AST process, remove the recursive decorator from the `decorator_list`.

I will choose the second solution, ensuring the recursive decorator is applied no matter where we call `foo` from.

```diff
        func_def_node = func_ast.body[0]

+        for d in func_def_node.decorator_list:
+            if d.id == decorator.__name__:
+                func_def_node.decorator_list.remove(d)
+                break

        expr_node = func_def_node.body[0]
```


### Filtering Out Builtins

Now, if you run the snippet code again, you will likely encounter the error: `TypeError: module, class, method, function, traceback, frame, or code object was expected, got builtin_function_or_method`.

To avoid this, we need to filter out the builtin functions from the AST modification process.

One simple way to do this is to return the function directly when we encounter a `builtins` function. You can modify the decorator in the following way:
```diff
+import types
...
def decorator(func):
    def wrapper(*args, **kwargs):
+        if isinstance(func, types.BuiltinFunctionType):
+            return func(*args, **kwargs)

        print(f"I'm decorating {func.__name__}")
```

### Iterating Over (Relevant) Nodes

It seems to run correctly, but there are still one or two significant issues.

If you run the snippet code, the result will be:
```text
I'm decorating foo

I'm bar
```
Which is definitely not what we want. And if you simply add a variable assignment, it is even worse!
```python
@decorator
def foo():
    x = 5
    print("I'm foo")
    bar()
```
**Error**: `AttributeError: 'Constant' object has no attribute 'func'`

The faulty line is `expr_node = func_def_node.body[0]`. We are accessing **only** the first node of the body. But in a real-world scenario, there are multiple assignments, calls, and so on. In the best case, we don't get an error, but we don't process every relevant node.

The solution is to iterate over every node and only process the ones of interest. So, instead of _assigning_ `expr_node = func_def_node.body[0]`, we will _iterate_ over `func_def_node.body`:
```python
for node in func_def_node.body:
    if not isinstance(node.value, ast.Call):
        continue

    node_func = node.value.func
    node.value = ast.Call(
        func=ast.Call(
            func=ast.Name(id=decorator.__name__, ctx=ast.Load()),
            args=[node_func],
            keywords=[]
        ),
        args=[],
        keywords=[]
    )
```

### Passing arguments
Everything runs smoothly now. But what if the inner function receives a parameter?

```py
def bar(x):
    print(f"I'm bar and have {x} hours in front of me.")

@decorator
def foo():
    x = 5
    print("I'm foo")
    bar(x)
```
**Error:** `TypeError: bar() missing 1 required positional argument: 'x'`

Just add the arguments in args:
```diff
            node_func = node.value.func
+           node_args = node.value.args
            node.value = ast.Call(
                func=ast.Call(
                    func=ast.Name(id=decorator.__name__, ctx=ast.Load()),
                    args=[node_func],
                    keywords=[]
                ),
+               args=node_args,
                keywords=[]
```

### Importing a Decorator from Another Module
If you define your decorator in a `decorator.py` module and want to use it in `main.py`, you may encounter an error like:
`NameError: name 'bar' is not defined`.

This error occurs because the decorator lacks knowledge of the context within the `main.py` module. 
It is aware of `foo` (the first function passed to the decorator) and `print` (since it is a builtin accessible from any module), but it knows nothing about `bar`.

The solution is to provide the context of `func` during the `exec` command, rather than the context of `decorator.py`:
```diff
         ast.fix_missing_locations(func_ast)
         module_compile = compile(func_ast, inspect.getsourcefile(func), "exec")
-        exec(module_compile, globals())
+        exec(module_compile, func.__globals__)
 
-        modified_func = globals()[func.__name__]
+        modified_func = func.__globals__[func.__name__]
 
         return modified_func(*args, **kwargs)
     return wrapper
```

### Calling a function twice

Remember when I mentioned we could simply do `func_def_node = func_ast.body[0]`?

Well, it turns out that it's not always `True`:
```python
@decorator
def foo():
    x = 5
    print("I'm foo")
    bar()
    bar()
```
When calling a function multiple times, it won't work. The source code from the second call of `bar` results in:
`from simple_draft import decorator\n\n\ndef baz():\n    print("I'm baz")\n`
Why? I'm not sure. But we have multiple possible solutions.

You could check if the node is a `FunctionDef` and return it directly. Since we already compiled and bound the function at its first call, it should be okay (and it is in this case):

```python
if not isinstance(func_def_node, ast.FunctionDef):
    return func(*args, **kwargs)
```

Or you could simply assign to `func_def_node` the first node that is an instance of:

```python
func_def_node = [node for node in func_ast.body if isinstance(node, ast.FunctionDef)][0]
```

I agree. Neither of those solutions seems satisfying enough. 

But don't worry, in the next chapter, we'll try to solve this using `ast.Transformer` and a node visitor.

[Part 3: AST Transformers](/blog/2024/04/19/recursive-decorator-Refinements-(part-3).html)

### simple_draft.py 

```python
import ast
import inspect
import types


def decorator(func):
    def wrapper(*args, **kwargs):
        if isinstance(func, types.BuiltinFunctionType):
            return func(*args, **kwargs)

        print(f"I'm decorating {func.__name__}")

        source_code = inspect.getsource(func)
        func_ast = ast.parse(source_code)
        # Option 2
        func_def_node = [node for node in func_ast.body if isinstance(node, ast.FunctionDef)][0]

        # Option 1
        # func_def_node = func_ast.body[0]
        # if not isinstance(func_def_node, ast.FunctionDef):
        #     return func(*args, **kwargs)

        for d in func_def_node.decorator_list:
            if d.id == decorator.__name__:
                func_def_node.decorator_list.remove(d)
                break

        for node in func_def_node.body:
            if not isinstance(node.value, ast.Call):
                continue

            node_func = node.value.func
            node_args = node.value.args
            node.value = ast.Call(
                func=ast.Call(
                    func=ast.Name(id=decorator.__name__, ctx=ast.Load()),
                    args=[node_func],
                    keywords=[]
                ),
                args=node_args,
                keywords=[]
            )

        ast.fix_missing_locations(func_ast)
        module_compile = compile(func_ast, inspect.getsourcefile(func), "exec")
        exec(module_compile, func.__globals__)

        modified_func = func.__globals__[func.__name__]

        return modified_func(*args, **kwargs)
    return wrapper


def bar(x):
    print(f"I'm bar and have {x} hours in front of me.")

@decorator
def foo():
    x = 5
    print("I'm foo")
    bar(x)
    bar(x)
	
```
```shell
> foo()
I'm decorating foo
I'm foo
I'm decorating bar
I'm bar and have 5 hours in front of me.
```