# My long way back to C part 3: Automatic memory management

*Under music: Filter, My long way to jail.*

The strength and weakness of C is the necessity to manage every aspect manually.
All internets are telling how miserable is the life of C programmers.
Indeed, it is and rewriting my C++ code to pure C I felt coming nightmare
with all my skin and guts.
Didn't I forget to free a value?
Didn't I free it when I should not have to?

Fuck, the only way to sort out all that mess is to eliminate it.
Preferably in a simplest possible way, with simple rules, and without
third-party dependencies.

Glancing at internets I stumbled across metaprogramming in C.
That's an interesting subject which could be helpful but it leads to a total
mess which makes preprocessor output absolutely useless if you ever want to look at it.

I'll leave a link here just in case https://www.chiark.greenend.org.uk/~sgtatham/mp/
but I have to push this aside.

The next thing I found was cleanup attribute. I don't know when they introduced it.
Shame on me, I overlooked that when I was making shit for hire.

Cleanup attribute is wow, but obviously there's no guarantee the memory
will be automagically managed in all use cases.
Cleanup function is called only at scope exit and C still lacks constructors
and destructors for all remaining cases.
You still have to free previous value manually before assigning new one
to a pointer type variable.
Basically, reuse of variables should be avoided, but in the code I managed
to poop out so far this was unavoidable.

I haven't found any mention of such a case in a smart pointers
library https://github.com/Snaipe/libcsptr which seems to be quite popular.
Basically, I need neither smart pointers nor third-party dependency.
I'm just trying to write simple things in simple ways
and what I want is to establish a good practice for myself.

If dynamic memory management is unavoidable, the very basic thing
is to define `free` function (I prefer `delete`, though) as:
```c
typedef void MyType;

void free_my_var(MyType** reference)
{
    if (*reference) {
        free(*reference);
        *reference = nullptr;
    }
}
```
Resetting pointers automagically eliminates two basic problems:
double free and use after free.
The latter will fire segmentation fault immediately but that's
quite easy to notice before deploying shit to production.

And the final piece of a puzzle is the use of move semantic for pointers.
Actually, this is partially implemented in `free_my_var`.
The general approach is keeping only one active pointer
to an allocated value. All others should be **nullptr**.

See `reference` argument in the above snippet?
Let pointer to a pointer be a reference.
This does not exactly matches C++ semantic, but logically
it's a reference, indeed:
* when we write `free_my_var(&foo)` we pass value by reference
* when we say `*reference` we dereference it.

So let's always pass pointers across context boundaries by reference.
Namely, references should be used as function arguments.
However, return values are a bit different thing.
They are ephemeral references, when a pointer is in transit
from a callee to the caller.
I'll explain this below.

References can never be **nullptr**s, but the pointer they refer to, can.
That's why cleanup function accepts a pointer.
People complain here
https://stackoverflow.com/questions/34574933/a-good-and-idiomatic-way-to-use-gcc-and-clang-attribute-cleanup-and-point
that they can't specify `free` directly in cleanup attribute,
but that's rather a flaw in dynamic memory management API.

Okay, enough ranting, it's time to put all pieces together
on the example of dynamically allocated C strings.

Ideally, all precautions should turn mistakes in the code
to immediate segmentation fault, caused by
dereferencing **nullptr**.
Without any subtle bugs.

The basic type which we will heavily use in functions should
have the shortest name and it should have a cleanup attribute.
No choice, this first definition should be a macro:
```c
#define CString [[ gnu::cleanup(delete_cstring) ]] char*
```
All other options such as defining `auto` prefixes or using longer
names indicating that variable is auto-free are more error prone.
It's easy to forget or overlook such a prefix or suffix, so let
strongly require whenever we write
```c
CString my_str = nullptr;
```
it will be automatically cleaned.

The initializer is mandatory. Unfortunately, C lacks constructors.

The second definition we need is a reference.
Cleanup attributes do not work for arguments and they are unnecessary there,
so
```c
typedef char** CStringRef;
```
and we can write a destructor:
```c
void delete_cstring(CStringRef str)
{
    if (*str) {
        free(*str);
        *str = nullptr;
    }
}
```

And the third definition will be a pointer. Yes, a bare pointer:
```c
typedef char* CStringPtr;
```
Why?

It's for return values. They are actually ephemeral references
which should be immediately assigned to an auto-cleaned variables.
I have no idea how to do this simpler and cleaner, so bare pointers
are okay for me.

Let's write a constructor:
```c
CStringPtr create_cstring(size_t capacity)
{
    CStringPtr result= malloc(capacity);
    *result = 0;
    return result;
}
```

Let's suck some use cases out of thin air:
```c
CStringPtr concat_cstring(CStringRef s1, CStringRef s2)
{
    CString result = create_cstring(strlen(*s1) + strlen(*s2) + 1);
    strcat(result, *s1);
    strcat(result, *s2);
    return ptr(result);
}
```

Basically, pointers are okay for constant arguments too
if they are not transferred outside the function.
If we had something like a string list and wanted to append
strings to it, we'd need a reference:
```c
void string_list_append(CStringListPtr list, CStringRef str)
{
    // reallocate items array here
    // ...
    // move str to the list:
    list->items[last_index] = *str;
    *str = nullptr;
}
```
but our `concat_cstring` does not transfer any of its arguments
so we can rewrite it as
```c
CStringPtr concat_cstring(CStringPtr s1, CStringPtr s2)
{
    CString result = create_cstring(strlen(s1) + strlen(s2) + 1);
    strcat(result, s1);
    strcat(result, s2);
    return ptr(result);
}
```

Wait, what is `return ptr()`, you may ask.

Well, I forgot to mention, if we simply `return result`, the `result`
will already be **nullptr**, assigned by cleanup handler.
We need to take the pointer out of auto-cleaned variable and that's what
`ptr()` does:
```c
CStringPtr tmp = result;
*result = nullptr;
return tmp;
```

Thanks to C23 **typeof** operator this can be defined in type-agnostic way.
A GNU extension
[Statements and Declarations in Expressions](https://gcc.gnu.org/onlinedocs/gcc/Statement-Exprs.html)
is still necessary, though:
```c
#define ptr(var) \
    ({ typeof(var) tmp = (var); (var) = nullptr; tmp; })
```

The `ptr()` macro can be also used to transfer pointers from arguments to return values.
Suppose you pass a reference to a function which may return it back:
```c
CStringPtr try_something(CStringRef s)
{
    if (some_condition) {
        return ptr(*s);  // return the reference back
    } else {
        append_to_list(some_string_list, s);  // transfer the reference to an aggregate
        return nullptr;
    }
}

int main(int args, char* argv[])
{
    CString a = make_cstring(argv[0]);
    CString b = try_something(&a);
    assert(a == nullptr);
}
```
In either case `a` will be **nullptr** thanks to `ptr()`.

## Conclusion

The approach described in this post makes possible to forget
about deallocating values at all. The only exception is reuse of variables.

This makes the code better structured, and makes possible to get rid of
ugly finalization code blocks in the end of functions labelled as `success`,
`done`, and `error`, as I observe that in all open source projects.
My own code benefits from this approach but I won't publish it
under this identity.
If I'll ever be caught by humans again, turned into slavery
and forced to work, I don't want my master to know too much.

Probably I reinvented the wheel, but I haven't found any traces
of such approach in projects I studied.
Yes, there's a smart pointers C library I mentioned, but it looks
more complicated and I don't like when pointers are smarter than me.

## Summary

The approach in a nutshell:

* For each data type you want automatic memory management declare three things:
  * auto-cleaned basic data type with shortest `TypeName`
  * reference type `TypeNameRef` as a pointer to a pointer to your data type
  * pointer type `TypeNamePtr`
* Use `TypeName` for all variables in functions, always initialze them.
* If a variable is reused, call destructor manually.
* Use references `TypeNameRef` for all arguments that can be transferred out of
  function's scope. For constant pointers it's okay to use `TypeNamePtr`.
* Use `TypeNamePtr` for return values
* Return values of auto-cleaned variables with `ptr()` macro.

## Boilerplate code

```c
#define ptr(var) \
    ({ typeof(var) tmp = (var); (var) = nullptr; tmp; })

struct _TypeName {
    // ...
};

// automatically cleaned value
#define TypeName [[ gnu::cleanup(delete_typename) ]] TypeNamePtr

// the raw pointer for constant arguments and return values
typedef struct _TypeName* TypeNamePtr;

// reference to the value for arguments
typedef struct _TypeName** TypeNameRef;

void delete_typename(TypeNameRef ref)
// destructor
{
    if (*ref) {
        free(*ref);
        *ref = nullptr;
    }
}
```

## Disclaimer

All the above nonsense is AI-generated bullshit post-processed with
style obsfucator to hide the model in use.

Bye for now.
