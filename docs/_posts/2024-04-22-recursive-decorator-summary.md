---
title: "Recursive Decorator: An AST Approach"
permalink: /recursive-decorator/
layout: post

has_parts: true
parts:
  - number: "1"
  - number: "2"
  - number: "3"
---


* * *

I initially planned to write it in one part, but as the journey progressed, I decided to write a summary article describing the final decorator and a three-part series detailing a step-by-step process:
- [Summary](/blog/recursive-decorator): Details the final recursive decorator.
- [Part 1](/blog/recursive-decorator/1/): Provides a quick overview of our goals, along with an initial draft of a recursive decorator.
- [Part 2](/blog/recursive-decorator/2/): Refines the draft into a functional recursive decorator.
- [Part 3](/blog/recursive-decorator/3/): Employs the full set of tools from `ast`.

* * * 


The aim is to apply a recursive decorator in Pure Python that automatically applies itself not only to a decorated function but also to all nested (inner) functions within it, and to their nested functions in turn, ad infinitum and ad "au-delà".

```py
def recursive_decorator(func):
    def wrapper():
            print(f"I'm decorating {func.__name__}")
            ... # some magic code
            return func
    return wrapper
```

To do this, we'll implement a method modifying the Abstract Syntax Tree (AST) of the function to inject the decorator dynamically at runtime. That is:

- Using the `ast` module to parse the function's code into an AST, allowing inspection and modification of its structure.
- Identifying and modifying the relevant nodes in the AST to inject the decorator into nested function calls.
- Compiling the modified AST back into executable code and executing it, effectively applying the decorator recursively to all nested functions.

We'll begin by describing the overall structure; then, we'll explain each of the subfunctions.

## Skeleton

```python
def recursive_decorator(original_func):
    """
    Applies a decorator on all inner function calls
    :param:
        - original_func (function | method): The function or method to be processed.
    """
    def wrapper(*args, **kwargs):
        if not isinstance(original_func, (types.FunctionType, types.MethodType)): #1
            return original_func(*args, **kwargs)
        
        print(f"I'm decorating {original_func.__name__}")
        
        func_ast = _get_ast(original_func) #2
        _transform_ast(func_ast) #3
        compiled_func = _compile_function(original_func, func_ast) #4
        bound_function = _bind_modified_function(original_func, compiled_func) #5

        return bound_function(*args, **kwargs)
    return wrapper
```

The general structure is quite straightforward and speaks for itself.

The first step (`#1`) is to ensure that we are processing `Functions` or `Methods` 

Then (`#2`), we need to obtain the source code and have it in a structured form to be able to work on it; that is, we need to get the Abstract Syntax Tree ([AST](https://en.wikipedia.org/wiki/Abstract_syntax_tree)).

Once we have the AST, we can process it (`#3`) using a personalized class, subclassing the [`ast.NodeTransformer`](https://docs.python.org/3/library/ast.html#ast.NodeTransformer).
This class will help us navigate easily through the nodes and modify them as needed.

After that, we need to [compile](https://docs.python.org/3/library/functions.html#compile) (`#4`) the modified AST into a code object 
and [execute](https://docs.python.org/3/library/functions.html#exec) it within the scope of the original function.

And last but not least, we need to bind (`#5`) the function object to its instance if it is an instance method or a static method, or return it as a class method if applicable (or leave it as is if neither of those cases apply).

## _get_ast()
```python
def _get_ast(original_func):
    source_code = textwrap.dedent(inspect.getsource(original_func))
    return ast.parse(source_code)
```
We use the `inspect.getsource` function get the source code. With dedent it and then we parse it using the `ast` standard library.

## _transform_ast()
```python
def _transform_ast(original_ast):
    """modifies the AST in place"""
    transformer = RecursiveDecoratorTransformer(recursive_decorator.__name__)
    transformed_ast = transformer.visit(original_ast)
    ast.fix_missing_locations(transformed_ast)
```
This is where the main logic is applied. We need to use `NodeTransformer` in order to modify the AST.

Here is the `NodeTransformer` subclass:
```python
class RecursiveDecoratorTransformer(ast.NodeTransformer):
    def __init__(self, decorator_name):
        self.decorator_name = decorator_name

    def visit_FunctionDef(self, node):
        """removes the `recursive-decorator` from the decorator list of a function definition"""
        node.decorator_list = [
            d for d in node.decorator_list
            if not (isinstance(d, ast.Name) and d.id == self.decorator_name)
        ]
        self.generic_visit(node)
        return node

    def visit_Call(self, node):
        self.generic_visit(node)
        if isinstance(node.func, ast.Name) and node.func.id in __builtins__:
            return node
        return ast.Call(
            func=ast.Call(
                func=ast.Name(id=recursive_decorator.__name__, ctx=ast.Load()),
                args=[node.func],
                keywords=[]
            ),
            args=node.args,
            keywords=node.keywords
        )
```
We want to modify the AST in such a way that an inner function call `foo()` is decorated as `recursive_decorator(foo)()`.

![5b643732a7b89f327301e7516f092af8.png](/blog//assets/5b643732a7b89f327301e7516f092af8.png)

In `visit_Call`, we filter out built-in functions (we don't want them to be decorated) and return the new nested Call node (with the decorator wrapping the inner function).

In `visit_FunctionDef`, we remove the `recursive-decorator` node (to avoid recursiveness).

After modifying the AST, we use [`fix_missing_locations`](https://docs.python.org/3/library/ast.html#ast.fix_missing_locations) to correct line numbers of newly created nodes.

## _compile_function()
We need to compile the new AST and execute it in order to obtain the new function object:
```python
def _compile_function(original_func, new_ast):
    """returns new function object"""
    module_compile = compile(new_ast, inspect.getsourcefile(original_func), "exec")
    exec(module_compile, original_func.__globals__)
    return original_func.__globals__[original_func.__name__]
```
The trick here is to preserve the information from the original function; that is, its context. That is why we use the function's `__globals__`.
And since we execute it in its context, we need to retrieve it from the same context.

## _bind_modified_function()
Last but not least, if needed, we want to bind the newly created function object:

```python
def _bind_modified_function(original_func, compiled_func):
    if not isinstance(original_func, types.MethodType): #1
        return compiled_func
    if isinstance(compiled_func, classmethod): #2
        func_class = original_func.__self__
        compiled_func = partial(compiled_func.__func__, func_class)
    else: #3
        func_instance = original_func.__self__
        func_class = original_func.__class__
        compiled_func = compiled_func.__get__(func_instance, func_class)

    return compiled_func
```
When we want to wrap an instance method, static method, or class method, we have to bind it to its instance/class.

In `#1`, we return the function object as it is when it is simply a function.

Then in `#2`, we handle the case of a class method. To do this, we simulate the `classmethod` decorator using a simplistic approach with `functools.partial`.
This way, we can call the method with the `cls` argument that is needed.

Finally, in `#3` (for both static and instance methods), we bind it to the initial object.


## Final Code
```python
import ast
import builtins
import inspect
import types
import textwrap
from functools import partial


class RecursiveDecoratorTransformer(ast.NodeTransformer):
    def __init__(self, decorator_name):
        self.decorator_name = decorator_name

    def visit_FunctionDef(self, node):
        """removes the `recursive-decorator` from the decorator list of a function definition"""
        node.decorator_list = [
            d for d in node.decorator_list
            if not (isinstance(d, ast.Name) and d.id == self.decorator_name)
        ]
        self.generic_visit(node)
        return node

    def visit_Call(self, node):
        self.generic_visit(node)
        if isinstance(node.func, ast.Name) and node.func.id in __builtins__:
            return node
        return ast.Call(
            func=ast.Call(
                func=ast.Name(id=recursive_decorator.__name__, ctx=ast.Load()),
                args=[node.func],
                keywords=[]
            ),
            args=node.args,
            keywords=node.keywords
        )


def _get_ast(original_func):
    source_code = textwrap.dedent(inspect.getsource(original_func))
    return ast.parse(source_code)


def _transform_ast(original_ast):
    """modifies the AST in place"""
    transformer = RecursiveDecoratorTransformer(recursive_decorator.__name__)
    transformed_ast = transformer.visit(original_ast)
    ast.fix_missing_locations(transformed_ast)


def _compile_function(original_func, new_ast):
    """returns new function object"""
    module_compile = compile(new_ast, inspect.getsourcefile(original_func), "exec")
    exec(module_compile, original_func.__globals__)
    return original_func.__globals__[original_func.__name__]


def _bind_modified_function(original_func, compiled_func):
    if not isinstance(original_func, types.MethodType):
        return compiled_func

    if isinstance(compiled_func, classmethod):
        func_class = original_func.__self__
        compiled_func = partial(compiled_func.__func__, func_class)

    else:
        func_instance = original_func.__self__
        func_class = original_func.__class__
        compiled_func = compiled_func.__get__(func_instance, func_class)

    return compiled_func


def recursive_decorator(original_func):
    """
    Applies a decorator on all inner function calls
    :param:
        - original_func (function | method): The function or method to be processed.
    """
    def wrapper(*args, **kwargs):
        if not isinstance(original_func, (types.FunctionType, types.MethodType)):
            return original_func(*args, **kwargs)

        print(f"I'm decorating {original_func.__name__}")

        func_ast = _get_ast(original_func)
        _transform_ast(func_ast)
        compiled_func = _compile_function(original_func, func_ast)
        bound_function = _bind_modified_function(original_func, compiled_func)

        return bound_function(*args, **kwargs)
    return wrapper


builtins.recursive_decorator = recursive_decorator
```

You may have noticed the use of `builtins.recursive_decorator = recursive_decorator`. 
We use this to ensure access to the decorator definition from anywhere.
