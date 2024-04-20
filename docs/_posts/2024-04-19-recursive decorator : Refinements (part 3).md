---
layout: page
title: Final recursive-decorator (Part 3)
#permalink: /about/
---
* * *

I initially thought to write it in one part, but it seemed too intense to do so. I divided it into three parts:
- [Part 1](/blog/2024/04/19/recursive-decorator-AST-Approach.html): provides a quick description of what we aim to achieve, along with a clunky draft of a recursive decorator.
- [Part 2](/blog/2024/04/19/recursive-decorator-troubleshooting-(part-2).html): corrects the draft to have a functional recursive decorator.
- [Part 3](/blog/2024/04/19/recursive-decorator-Refinements-(part-3).html) (work in progress): utilizes the full set of tools from `ast` to make it even more beautiful.

* * *

# Using `ast.NodeTransformer`

To be written...

Changed:
* add the NodeTransformer
* add the keywords argument
* only accept `types.FunctionType`
* dedent with `textwrap` (class methods have an indent)

[//]: # (https://stackoverflow.com/questions/56479418/indentationerror-during-ast-parse-and-ast-walk-of-a-function-which-is-a-meth)



## Results
With the same amount of lines, we achieve more readable and more robust code:
```python
import ast
import inspect
import types
import textwrap


class RecursiveDecoratorTransformer(ast.NodeTransformer):
    def __init__(self, decorator_name):
        self.decorator_name = decorator_name

    def visit_FunctionDef(self, node):
        node.decorator_list = [d for d in node.decorator_list if not (isinstance(d, ast.Name) and d.id == self.decorator_name)]
        self.generic_visit(node)
        return node

    def visit_Call(self, node):
        if isinstance(node.func, ast.Name) and node.func.id not in __builtins__:
            return ast.Call(
                func=ast.Call(
                    func=ast.Name(id=decorator.__name__, ctx=ast.Load()),
                    args=[node.func],
                    keywords=[]
                ),
                args=node.args,
                keywords=node.keywords
            )
        return self.generic_visit(node)


def decorator(func):
    def wrapper(*args, **kwargs):
        if not isinstance(func, types.FunctionType):
            return func(*args, **kwargs)

        print(f"I'm decorating {func.__name__}")

        # Getting the AST from the source code
        source_code = textwrap.dedent(inspect.getsource(func))
        func_ast = ast.parse(source_code, mode='exec')

        # Transforming the AST
        transformer = RecursiveDecoratorTransformer(decorator.__name__)
        transformed_ast = transformer.visit(func_ast)
        ast.fix_missing_locations(transformed_ast)

        # Compilation and execution
        module_compile = compile(transformed_ast, inspect.getsourcefile(func), "exec")
        exec(module_compile, func.__globals__)

        # Retrieving the modified function from the context
        modified_func = func.__globals__[func.__name__]

        return modified_func(*args, **kwargs)
    return wrapper

```
