# My long way back to C part 1: Tears, snot, and sorrow

*Under music: Filter, My long way to jail.*

They are late. Those who start learning python.
It was cool 20 years ago. It was decent 10 years ago.
Today everything is rotten and all that shit they brought
from statically typed languages makes the code less and less
readable.
Their zen is no longer relevant.

I had to escape. It was not easy, though. Python made me lazy and depraved.
It took a lot of efforts to make up my mind and find a way out.

But what's the replacement? I can't say modern languages scare me.
They are simply have no single genious seed inside.
Ad-hoc shit is not what I want to dip my toe in.

So I turned to C++ and started writing a simple parser in it.
C++ is cool, and here are just a few points that make it cool:
* auto keyword
* C++17 structured bindings
* range-based for loop
* RAII

Nevertheless, very quickly I felt a strong desire to puke
and realized that C++ still sucks:
* String slicing: `StringView` sucks.
* Unicode: they ran the full circle and now look like a snake eating its tail.
  They dropped that stuff from `std` but haven't wiped splashes of shit so far.
  So now it's not even possible to convert UTF-8 to a standard `u8` type
  without a third-party ICU.
* Move semantic is not automagic for arguments declared as `&&`. Maybe, there
  are reasons for that but it's easy to forget `std::move`.
* Why it's so fucking difficult to overload comparison operator?
  What type qualifiers I should care about and why `const` fixes the problem?
  Kinda shitmagic I have no least desire to learn.
* Three way comparison sucks. Can't do that `<=>` for `int` and floating point types
  because of different result type. Yuck!
* `.at()` method sucks in all sense. A safe operator should be less verbose, IMAO,
  and that's what `[]` is for. More verbose `.at()`  can be quick, dirty, and unsafe.
  However on the C++ planet they have to write `a["foo"].at(3).["bar"]`.
* chrono operartor / is a total shit too. Who hardcodes dates?
  They write simple scripls in it?
  I agree, `bash` sucks, but using C++ as a replacement... wow.
  Anyway, `/` is division!
  Basically, misusing operatrors is the same old song.
  They are still happy with `<<` and `>>` for streams.
* `any` type was too difficult to comprehend for me, so I started implementing my own.
  And I was not alone, glancing at similar projects and even at source code of libraries
  installed in my system.
* Finally, the way I had to define a hash function for my data type led to a messy code
  with weird order of private and public sections.

I started rewriting it in pure C. That had the immediate result: the include file became
much more straightforward and readable. But I realized I need an underlying thing:
[arena allocator](20240729-arena-allocator.md).
