
* * *

I initially thought to write it in one part, but it seemed too intense to do so. I divided it into three parts:
- [Part 1](/blog/2024/04/19/recursive-decorator-AST-Approach.html): provides a quick description of what we aim to achieve, along with a clunky draft of a recursive decorator.
- [Part 2](/blog/2024/04/19/recursive-decorator-troubleshooting-(part-2).html): corrects the draft to have a functional recursive decorator.
- [Part 3](/blog/2024/04/19/recursive-decorator-Refinements-(part-3).html) (work in progress): utilizes the full set of tools from `ast` to make it even more beautiful.

* * *

# Using `ast.Transformers` and Nodes Visitors

This is great for learning. But we could simplify the decorator and make it more readable by using the full set of tools from the `ast` library.
[WIP]
