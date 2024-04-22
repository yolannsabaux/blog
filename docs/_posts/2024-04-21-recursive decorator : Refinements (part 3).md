---
title: "Recursive Decorator: An AST Approach (Part 3)"
permalink: /recursive-decorator/3/
layout: post
---

* * *

I initially thought to write it in one part, but it seemed too intense to do so. I divided it into three parts:
- [Part 1](/blog/recursive-decorator/1/): provides a quick description of what we aim to achieve, along with a clunky draft of a recursive decorator.
- [Part 2](/blog/recursive-decorator/2/): corrects the draft to have a functional recursive decorator.
- [Part 3](/blog/recursive-decorator/3/): utilizes the full set of tools from `ast` to make it even more beautiful.

* * *

In Part 2 of this series, we troubleshoot some basic issues.

Rest assured, there are many others that we can find! Some of them will be fixed, while others will not be supported (e.g., assignment of lambda functions).

The goal of this chapter is to make the use of the decorator possible in the following situations:

#### Test Cases
```python
from recursive_decorator import decorator

class Foo:
    x = 5
    @classmethod
    def class_method(cls):
        print(f"I'm {class_method.__name__} Instance of {Foo.__name__}")
        baz(cls.x)

    @staticmethod
    def static_method():
        print(f"I'm {static_method.__name__} Instance of {Foo.__name__}")
        baz(5)

    def instance_method(self):
        print(f"I'm {instance_method.__name__} Instance of {Foo.__name__}")
        baz(self.x)

        
def qux():
    print(f"I'm {qux.__name__}")
    return 10

def baz(x=1):
    print(f"I'm baz {x}")

def bar():
    x = 5
    print(f"I'm {bar.__name__}")
    baz(x)

@decorator
def foo():
    print(f"I'm {foo.__name__}")
    y = bar
    y()
    bar()
    baz(x=qux())
    f = Foo()
    f.instance_method()
    f.static_method()
    f.class_method()
    Foo.class_method()
```
```text
> foo()
I'm decorating foo
I'm foo
I'm decorating bar
I'm bar
I'm decorating baz
I'm baz 5
I'm decorating bar
I'm bar
I'm decorating baz
I'm baz 5
I'm decorating qux
I'm qux
I'm decorating baz
I'm baz 10
I'm decorating instance_method
I'm instance_method Instance of Foo
I'm decorating baz
I'm baz 5
I'm decorating static_method
I'm static_method Instance of Foo
I'm decorating baz
I'm baz 5
I'm decorating class_method
I'm class_method Instance of Foo
I'm decorating baz
I'm baz 5
```

To do that, we'll make our life a bit easier with the use of `ast.NodeTransformer`.

# Using `ast.NodeTransformer`

## What is it?

We had to write a lot of boilerplate code to iterate over the nodes, check if the node is an instance of the class node that we want, and so on.

`ast` provides multiple tools to avoid this. For instance, if you just want to walk the tree, you could subclass `ast.NodeVisitor`.

In our case, we not only want to walk the tree but also modify it. For this purpose, we can subclass `ast.NodeTransformer`.
It will allow us to crawl the nodes of the AST and modify them in the process.

```python
import ast

class RecursiveDecoratorTransformer(ast.NodeTransformer):
    pass
```

Then you just have to define your node-specific visitor methode following the `visit_<NodeType>` pattern
and define how you want to process the node.

```python
class RecursiveDecoratorTransformer(ast.NodeTransformer):
    def visit_FunctionDef(self, node):
        pass # Magic code applying on a `FunctionDef` node 
        return node  
```
See where we are going?

**One last thing**

There is one trick. If you modify a node that has child nodes and want to also crawl those child nodes, you will have to call a specific method before returning the node: `self.generic_visit(node)` (more on this later).


## Refactor

Let's refactor bit by bit our code

### Delete our decorator from the `decorator_list`

We had
```py
func_def_node = [node for node in func_ast.body if isinstance(node, ast.FunctionDef)][0]
for d in func_def_node.decorator_list:
    if d.id == decorator.__name__:
        func_def_node.decorator_list.remove(d)
        break
```

Now we can do

```python
# class definition
def __init__(self, decorator_name):
    self.decorator_name = decorator_name
        
def visit_FunctionDef(self, node):
    node.decorator_list = [ d for d in node.decorator_list if not d.id == self.decorator_name]
    `self.generic_visit(node)`  # We want to crawl the rest of the tree. Otherwise, it would stop there.
    return node
```
Just like that!

When instantiating the class, just pass the `decorator_name` as a parameter: `RecursiveDecoratorTransformer(decorator.__name__)`

### Iterate over nodes (ast.Call)
As we said, there is no need to iterate and check over the nodes.

```python
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
```

Becomes
```python
# class definition
def visit_Call(self, node):
    self.generic_visit(node)
    return ast.Call(
        func=ast.Call(
            func=ast.Name(id=decorator.__name__, ctx=ast.Load()),
            args=[node.func],
            keywords=[]
        ),
        args=node.args,
        keywords=node.keywords
    )
```
Similarly, we call `self.generic_visit(node)`; You could have situations with nested calls such as in `baz(qux())`.

This supports both `FunctionType` and `MethodType`. 
The difference between the two is handled thanks to `node.func`. 
It will be an `ast.Name` for `FunctionType` and an `ast.Attribute` for `MethodType`.

### Check for Builtins

In the first version, we were checking if the function was a Builtin before processing it.
```python
if isinstance(func, types.BuiltinFunctionType):
    return func(*args, **kwargs)
```
We were applying the decorator to the built-in function, and when it was called, we were stopping the process.

A better way is to filter out the builtins directly in the `visit_Call`:
```python
#def visit_Call(self, node):
#    self.generic_visit(node)
    if isinstance(node.func, ast.Name) and node.func.id in __builtins__:
        return node
#    return ast.Call(
```
We just added an extra check to only check functions (`ast.Name`) and not methods (`ast.Attribute`).

### Keywords arguments

Maybe you saw it, but in addition to the aforementioned <sup>(beautiful word)</sup> changes, `keywords` has been added to the new `ast.Call`.

```python
def visit_Call(self, node):
# ...
#    return ast.Call(
#        func=ast.Call(
#            func=ast.Name(id=decorator.__name__, ctx=ast.Load()),
#            args=[node.func],
#            keywords=[]
#        ),
#        args=node.args,
        keywords=node.keywords
#    )
```
It solves the problem we would have with keyword arguments such as `baz(x=qux())`.


### Filtering out instantiation

Take a look at the following AST:
```text
>>> get_ast("""
... f = Foo()
... """)
Module(
    body=[
        Assign(
            targets=[
                Name(id='f', ctx=Store())],
            value=Call(
                func=Name(id='Foo', ctx=Load()),
                args=[],
                keywords=[]))],
    type_ignores=[])
```
Function calls or class instantiation are basically the same (in the AST; in the CPython interpreter, it is different).

And in this case, we don't want to crawl the child node. 
We want to avoid: `f = A()` -> `f = decorator(A)()`

But I am still unsure what the best way is. You have two solutions:

#### Whitelist: Complete the NodeTransformer Class
```python
def visit_Assign(self, node):
    return node
```
This will return the node directly without visiting its children.

#### Blacklist: Directly in the Wrapper

```python
#def wrapper(*args, **kwargs):
    if not isinstance(func, (types.FunctionType, types.MethodType)):
        return func(*args, **kwargs)
```
Blacklisting means that we only process functions and methods. 
It's dirtier:
- The filtering responsibility belongs to the NodeTransformer.
- This situation occurs: `f = A()` -> `f = decorator(A)()`,

but there might be corner cases that are not covered.


## You said classes?

If you run the modified decorator with the "Test Cases" file containing classes, you will get two errors.

### Indentation
```python
f = Foo()
f.instance_method()
```
Will raise an error: "IndentationError: unexpected indent".

The same applied if you call `get_ast` with:

```text
>>> get_ast("""
...     def class_method(cls):
...         print(f"I'm {class_method.__name__} Instance of {Foo.__name__}")
...         baz(cls.x)
... """)
Traceback (most recent call last):
...
IndentationError: unexpected indent
```

The solution: just use the `textwrap` library
```python
import textwrap
...
source_code = textwrap.dedent(inspect.getsource(func))
```

### James Bind
This one was trickier to solve. Run your code and you will get:
`TypeError: instance_method() missing 1 required positional argument: 'self'`

I tried many things (mainly out of despair). I even tried to take the original function and inject the code object of the modified function<sup>*(1)</sup>.

Actually, the solution is quite simple (when you know it).

If you run the following:
```python
class A:
    def foo(self):
        print("I'm Foo")

def decorator(func):
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)  # break here
    return wrapper

if __name__ == '__main__':
    f = A()
    decorator(f.foo)()
```
and breaks on the `return func`, the object you get is:
```
<bound method A.foo of <__main__.A object at 0x721c52bc3730>>
```


But if you break on the `return modified_func(*args, **kwargs)` (from our decorator) when processing a method, you will get:

```
<function instance_method at 0x7b61c7f8f5b0>
```

From there, you know that you need to bind the function to the instance.<sup>*(2)</sup>:

```python
# bind modified class method to instance
if isinstance(func, types.MethodType):
    instance = func.__self__
    modified_func = modified_func.__get__(instance)
```

* * *
<sup>*(1) You cannot modify the code object of a function. You can only create a new object from its code arguments with `CodeType`.
But it is still a new object with no binding.</sup>

<sup>*(2) I found the `__get__()` solution in the "Python Cookbook" (Beazley, Jones - 3rd edition) in the "9.9 Defining Decorators As Classes" recipe.</sup>


## Results
With (almost) the same amount of lines, we achieve more readable and more robust code:

```python
import ast
import inspect
import types
import textwrap


class RecursiveDecoratorTransformer(ast.NodeTransformer):
    def __init__(self, decorator_name):
        self.decorator_name = decorator_name

    def visit_Assign(self, node):
        """
        In the case of a class instantiation, we don't want to crawl the children node
        We want to avoid:
        `f = A()` -> `f = decorator(A)()`
        """
        return node

    def visit_FunctionDef(self, node):
        node.decorator_list = [d for d in node.decorator_list if not d.id == self.decorator_name]
        self.generic_visit(node)
        return node

    def visit_Call(self, node):
        self.generic_visit(node)
        if isinstance(node.func, ast.Name) and node.func.id in __builtins__:
            return node
        return ast.Call(
            func=ast.Call(
                func=ast.Name(id=decorator.__name__, ctx=ast.Load()),
                args=[node.func],
                keywords=[]
            ),
            args=node.args,
            keywords=node.keywords
        )


def decorator(func):
    def wrapper(*args, **kwargs):
        print(f"I'm decorating {func.__name__}")

        # Getting the AST from the source code
        source_code = textwrap.dedent(inspect.getsource(func))
        func_ast = ast.parse(source_code)

        # Transforming the AST
        transformer = RecursiveDecoratorTransformer(decorator.__name__)
        transformed_ast = transformer.visit(func_ast)
        ast.fix_missing_locations(transformed_ast)

        # Compilation and execution
        module_compile = compile(transformed_ast, inspect.getsourcefile(func), "exec")
        exec(module_compile, func.__globals__)

        # Retrieving the modified function from the context
        modified_func = func.__globals__[func.__name__]

        # bind modified method to instance
        if isinstance(func, types.MethodType):
            instance = func.__self__
            modified_func = modified_func.__get__(instance)

        return modified_func(*args, **kwargs)
    return wrapper
```

## The beginning of the journey

Ah oui?
Yes, there are still lots of improvements that can be made:
- Debugging is a nightmare. We are creating all new objects with the `co_lineno` that are not correct (even if corrected thanks to `ast.fix_missing_locations`).
A solution would be to take the code arguments of the `original_func`, take the `__code__` of the modified function, and create a new object from it (see [original idea](https://blog.elishalom.com/tag/ast/)).
- ASTs don't contain information about comments. A solution would be to use a CST (Concrete Syntax Tree), but there are no tools in the Standard Library, and I wanted to do this in "Pure Python" (See [LibCST](https://libcst.readthedocs.io/en/latest/)).
- What about `functools.wraps`? It's THE decorator of decorators. 
Since we are creating new objects from the AST, there is no metadata to be preserved. Might be useful if the first points are implemented.

If you are reading those lines, ça veut dire que tu me connais. Si c'est toi Hugo, je te paye une bière.
Byeeee!

Ah yes, this code does not work for class method:
```python
f.class_method()
Foo.class_method()
```

Will you find why? 
Cheers