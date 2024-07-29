# My long way back to C part 2: Arena allocator

*Under music: Filter, My long way to jail.*

Every kid knows what arena allocator is. When I was a kid I knew that too,
but modern languages displaced that with garbage collection knowledge.
That's how garbage breaks into weak minds.

Arena allocator is so simple that every self-respecting programmer implements
their own version.
I glanced at a few on github but none of them suits my needs.
So I began implementing the best one.
Not because of NIH syndrome, just to recall how to write in C.

When I ran my first test I got segmentation fault.
That was anticipated, especially for a custom memory allocator.
But what caused this? On reflection I'd highlight three major points:
* careless reading `mmap` man page
* mistakes in bitwise expressions
* complex conditions

When writing in C I always resist the temptation to optimize for speed from
the very beginning. That's a curse of knowledge: my first language was
assembler and I still know things that are better not to know.
That may sound trite, but the key to bug-free code is its right structure,
when dangerous and error-prone operations are localized and it's easy
to understand at a glance *how* particular function does things, whereas
*what* the function does is self-explanatory by its name and arguments.

However, the latter is not always possible in C.
Remember `calloc`? It takes number of elements and size of each element.
Which means if you invoke that function you rather say *how* to allocate,
not *what* to allocate, i.e.
```c
int* array = calloc(100, sizeof(int));
```
You might argue, and I agree, the difference between *how* and *what*
is subtle, but the man page says:

> The calloc() function allocates memory for an array

Do you see? It's *memory for an array*, not *an array*.

In my allocator the difference between *how* and *what* becomes evident
because I want values to be properly aligned so I had to add alignment
parameter.
That makes invocation more cumbersome:
```c
int* array = arena_alloc(my_arena, 100 * sizeof(int), alignof(int));
```

Luckily, C has preprocessor which greatly simplifies the above
and, what is more important, it changes *how* semantic to *what*:
```c
int* array = arena_alloc(my_arena, 100, int);
```
What could be better?

The layout of the source file is well structured:

* General purpose helper functions and macros:
  you can copy-paste them to other projects without modifications.
* Arena and Region structures: that's what we'll work with.
* Internal functions: all you need to implement the final part,
* Public API

[Enjoy](https://github.com/amateur80lvl/arena),
if you can.

P.S. There's no any tests. Testing is for cowards and for those
who lacks self-confidence.
Here's a snippet I used to admire how allocator works:

```c
#include <stdio.h>
#include "arena.h"

int main(int argc, char* argv[])
{
    Arena *arena = create_arena(100);
    arena_print(arena);

    putchar('\n');

    for(int i = 0; i < 500; i++) {
        arena_alloc(arena, 3, long long);
    }

    arena_fit(arena, 10, char);
    arena_fit(arena, 3000, char);

    arena_print(arena);
    delete_arena(arena);
}
```
