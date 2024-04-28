---
title: "Recursive Decorator: An AST Approach (Part 1)"
permalink: /recursive-decorator/1/
redirect_from:
  - /2024/04/19/recursive-decorator-AST-Approach.html
layout: post

is_part: true
---


* * *

I initially planned to write it in one part, but as the journey progressed, I decided to write a summary article describing the final decorator and a three-part series detailing a step-by-step process:
- [Summary](/blog/recursive-decorator): Details the final recursive decorator.
- [Part 1](/blog/recursive-decorator/1/): Provides a quick overview of our goals, along with an initial draft of a recursive decorator.
- [Part 2](/blog/recursive-decorator/2/): Refines the draft into a functional recursive decorator.
- [Part 3](/blog/recursive-decorator/3/): Employs the full set of tools from `ast`.

* * * 

# TL;DR

The aim is to apply a recursive decorator in Python that automatically applies itself not only to a decorated function but also to all nested (inner) functions within it, and to their nested functions in turn, ad infinitum and ad "au-delà".

```py
def recursive_decorator(func):
    def wrapper():
            print(f"I'm decorating {func.__name__}")
            ... # some magic code
            return func
    return wrapper

def baz():
    print("I'm baz")

def bar():
    print("I'm bar")
    baz()
        
@recursive_decorator
def foo():
    print("I'm foo")
    bar()
```

```text
> foo()
I'm decorating foo
I'm foo
I'm decorating bar
I'm bar
I'm decorating baz
I'm baz
```

To do this, I'll implement a method modifying the Abstract Syntax Tree (AST) of the function to inject the decorator dynamically at runtime. That is:

- Using the `ast` module to parse the function's code into an AST, allowing inspection and modification of its structure.
- Identifying and modifying the relevant nodes in the AST to inject the decorator into nested function calls.
- Compiling the modified AST back into executable code and executing it, effectively applying the decorator recursively to all nested functions.


### Result `simple_draft.py`

```py
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
```

# Rough Design

## How to apply a decorator

In Python, decorators can be applied using the `@` symbol

```py
@recursive_decorator
def bar():
    pass
```

or by wrapping the function with the decorator manually:

```py
def bar():
    pass
    
recursive_decorator(bar)
```

But we need to find, at runtime, all the inner functions, decorate them, and then recursively decorate all their inner functions. How can we do that?

## Using AST

An AST (Abstract Syntax Tree) is a data tree representing the structure of a piece of code where each node corresponds to an element of the code, such as an expression, statement, function, etc.

With tools provided by the Python Standard Library, we can identify the nodes we want to modify and inject our `recursive-decorator`.

### Peek into the AST

To view the AST, this code snippet can be used:

```python
import ast
def get_ast(code):
    print(ast.dump(ast.parse(code), indent=4))
```

```
get_ast("""
@decorator
def foo():
    bar()
""")
```

```
Module(
    body=[
        FunctionDef(
            name='foo',			# This is our main function
            args=arguments(
                posonlyargs=[],
                args=[],
                kwonlyargs=[],
                kw_defaults=[],
                defaults=[]),
            body=[
                Expr(
                    value=Call(
                        func=Name(id='bar', ctx=Load()), # This is our inner function
                        args=[],
                        keywords=[]))],
            decorator_list=[
                Name(id='decorator', ctx=Load())])], # This is our decorator
    type_ignores=[])

```

#### Simplified Graph

![df65e4b4919e7cced360e618f14d0d77.png](/blog//assets/df65e4b4919e7cced360e618f14d0d77.png)

## What would the modified AST look like?

Remember that we need to decorate the inner function

```py
def foo():
    decorator(bar)
```

An easy way to do so is to inject into the `Call` Node another `Call` Node with the `decorator` as the `func` and `bar` as an argument, such as:
```diff
            body=[
                Expr(
                    value=Call(
-                        func=Name(id='bar', ctx=Load()),
+                        func=Call(
+                            func=Name(id='decorator', ctx=Load()),
+                            args=[
+                                Name(id='bar', ctx=Load())],
+                            keywords=[]),
-            decorator_list=[
-                Name(id='decorator', ctx=Load())])],
+            decorator_list=[])],
                       args=[],
                       keywords=[]))],
    type_ignores=[])
```

#### Old graph on the left and new graph on the right

![5b643732a7b89f327301e7516f092af8.png](/blog//assets/5b643732a7b89f327301e7516f092af8.png)

* In green: the new `Call` node with the decorator.
* In blue: the node containing the `bar` function.
* In red: the deleted decorator of the main function to avoid cyclical decoration.

# First Draft

## Simple Decorator

We need to return the modified function definition. A simple decorator that returns the function would be:

```py
def decorator(func):
    def wrapper(*args, **kwargs):
        print(f"I'm decorating {func.__name__}")
        # we'll put the magic code in here
        return func(*args, **kwargs)
    return wrapper

@decorator
def foo():
    print("I'm foo")
```

```text
> foo()
I'm decorating foo
I'm foo
```

## Get the AST object

We need the AST object. We can use `inspect` to get the source code as a string and use `ast` to parse it

```py
import inspect
import ast

func = bar
source_code = inspect.getsource(func) #The `inspect` can get us the source at runtime
func_ast = ast.parse(source_code) # `ast.parse` gets us the `ast` object
```

And the same way as before, we can dump it to visualize the AST:

```
> ast.dump(func_ast, indent=4)
Module(
# ...
```

## Modify the AST

#### Get the `FunctionDef` node

Remember the basic structure:

```
Module(
    body=[
        FunctionDef(
        ...
```

We only need the function definition of `foo`. In other words, what is inside of `Module.body[0]`:

```python
func_def_node = func_ast.body[0]
```

#### Get the `Expr` node

Now we have the `FunctionDef` node:

```
        FunctionDef(
        # ...
            body=[
                Expr(
                    value=Call(
                        func=Name(id='bar', ctx=Load()), # This is our inner function
                        args=[],
                        keywords=[]))],
            decorator_list=[
                Name(id='decorator', ctx=Load())])], # This is our decorator
    type_ignores=[])
```

From there we can extract the `Expr` node we want to modify

```python
expr_node = func_def_node.body[0].value
```

```text
Expr(
    value=Call(
        func=Name(id='print', ctx=Load()),
        args=[
            Constant(value="I'm foo")],
        keywords=[]))
```

#### Get the `func`

From there we can extract the functions and its args

```py
node_func = expr_node.value.func
```

We will use it to inject it as a parameter of our new `Call` node containing the `decorator` function.

### Create the New Node

We need to replace the `Call` node with a new one. We can again use `ast` to create a node from scratch by passing it the arguments we need, that is another `Call` node which will call the `decorator` and use `bar` as a parameter:
```python
        expr_node.value = ast.Call(
            func=ast.Call(
                func=ast.Name(id=decorator.__name__, ctx=ast.Load()), # the id is the name of the function
                args=[node_func], # node_func is the func we want to wrap with decorator
                keywords=[]
            ),
            args=[],
            keywords=[]
        )
```

### Compile Everything (Or Where the Magic Happens!)

Now that we have modified our AST, we need a way to make it understandable for the CPython interpreter. To do so, we need to compile it:
```py
        ast.fix_missing_locations(func_ast) 	# 1
        module_compile = compile(func_ast, inspect.getsourcefile(func), 'exec') 	# 2
        exec(module_compile, globals()) 	# 3

        modified_func = globals()[func.__name__] 	# 4
```

1. `fix_missing_locations` is used to fix the line numbers and column offsets that have changed following our AST modification; [documentation](https://docs.python.org/3/library/ast.html#ast.fix_missing_locations).
2. Compiles a source (normal string, a byte string, or an AST object) into a code object. The filename of the function definition. And `exec` is the mode; [documentation](https://docs.python.org/3/library/functions.html#compile).
3. The `exec` will bind (in the first iteration) our main function `foo` with a `code` object (`module_compile`) within a certain context (`globals()`) <sup>*(1)</sup>.
4. `exec` returns `None`. Therefore, we need to retrieve it directly where it is defined: in the `globals()`.

And voilà!

```py
def decorator(func):
    def wrapper(*args, **kwargs):
        print(f"I'm decorating {func.__name__}")

        source_code = inspect.getsource(func)
        func_ast = ast.parse(source_code)
        func_def_node = func_ast.body[0]

        expr_node = func_def_node.body[0]
        node_func = expr_node.value.func
        expr_node.value = ast.Call(
            func=ast.Call(
                func=ast.Name(id=decorator.__name__, ctx=ast.Load()),
                args=[node_func],
                keywords=[]
            ),
            args=[],
            keywords=[]
        )

        ast.fix_missing_locations(func_ast)
        module_compile = compile(func_ast, inspect.getsourcefile(func), "exec")
        exec(module_compile, globals())

        modified_func = globals()[func.__name__]

        return modified_func(*args, **kwargs)
    return wrapper
```

* * *

<sup>*(1) The differences between a code object, a function object, and a frame object can be roughly summarized as follows:</sup>
<sup>- code: the most primitive form (bytecode: a string of ones and zeros)</sup>
<sup>- function: contains a code object and an environment -> static</sup>
<sup>- frame: contains a code object and an environment -> at runtime (runtime representation of a function)</sup>

* * *

Well, not really "voilà". There are still some basic issues we still need to solve
<a class="post-link" href="/blog/2024/04/19/recursive-decorator-troubleshooting-(part-2).html">
Recursive decorator : Troubleshooting (part 2)
</a>