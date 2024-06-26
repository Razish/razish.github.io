---
title: "I Did A Learn - D Lang"
date: "2024-06-04"
tags: idal, dlang, d, programming, language, til, learn
---

## Starting with a blank slate

D is cute. I've seen some D snippets every now and then, but never looked into it. It seems powerful and stays out of your way.  
I like compiled languages (long live C 😍). It's been a long time since I last looked at it, and it seems to be a mature language by now.  
It seems to fit a similar niche to Go. Is Go simply _better_? Maybe it has a better concurrency model? Stronger corporate backing?  
That doesn't mean D is useless. Let's see how it _feels_!  
There's a (very handy, I assume) [DLang Tour](https://tour.dlang.org/) available. I don't want to use that until I absolutely must - a key priority I want to evaluate is "how easy is this to just pick up and hit the ground running?". If I had to write a tool in D tomorrow, could I? I can think of one way to find out...!

## Managing Expectations

I am dipping my toes into the waters of D and writing down the first thing that comes to mind. This is not an in-depth, exhaustive or professional review of the D language. It's an anecdotal journal of clumsily smashing things together over a weekend. I ended up writing most of this post before and after writing the code. I'm not sure how much fun it would be watching me rewrite the same bit of code multiple times and debug a tokeniser with my pea-sized brain 🙃 maybe next time.  
I am obviously not an expert on D. I might say some things that are factually incorrect. Do your own research and learning :)

## Pre-flight check

I imagine we need a compiler, build system, debugger, what else?  
`dub` seems to be the build system. a la Cargo, npm.

```console
$ brew install dub
# ... some stuff happened ...

$ dub

No package manifest (dub.json or dub.sdl) was found in
/Users/Razish/projects/dplayground
Please run DUB from the root directory of an existing package, or run
"dub init --help" to get information on creating a new package.

Error No valid root package found - aborting.
```

Cool, that was painless. And it gave us some helpful info. I'm gonna `dub init` my project and be on my way 🙃

```console
$ dub init
# ... selecting some things in the nice wizard ...
     Success created empty project in /Users/Razish/projects/dplayground
             Package successfully created in .
$ tree -a
.
├── .gitignore
├── dub.json
└── source
    └── app.d

$ cat source/app.d
import std.stdio;

void main()
{
        writeln("Edit source/app.d to start your project.");
}
```

Beautiful. I have a source file, project configuration and it's even configured git for me. Let's build it!

```console
$ dub build
Error Executable file not found: dmd

$ brew install dmd
dmd: The x86_64 architecture is required for this software.
Error: dmd: An unsatisfied requirement failed this build.
```

`dmd` (Digital Mars D Compiler) is the flagship compiler. Oops, there's no official binary for aarch64 🤦  
A little birdy told me there's another blessed option that _does_ work, `ldc`. It's built on LLVM. Let's go!

```console
$ brew install ldc
# ... some stuff happened ...

$ dub build
    Starting Performing "debug" build using ldc2 for aarch64, arm_hardfloat.
    Building dplayground ~master: building configuration [application]
     Linking dplayground

$ ./dplayground
Edit source/app.d to start your project.
```

Not only did `ldc` work, but `dub` picked it up automatically and _just used it_. This feels nice and not astonishing.  
Note, I'm going to run `dub` with the `-q` flag from here on, which just silences the build output.

One last thing. Linting. DScanner is the tool of choice, and it is _fast_ 🤩

```console
$ brew install dscanner
# ... some stuff happened ...

$ dscanner lint
./source/parse/entities.d(6:8): Warning: Public declaration 'Entity' is undocumented. (undocumented_declaration_check)
struct Entity
       ^^^^^^
./source/parse/entities.d(11:27): Warning: Parameter entities is never used. (unused_parameter_check)
string serialise(Entity[] entities)
                          ^^^^^^^^
./source/parse/entities.d(20:10): Warning: Variable bytes is never used. (unused_variable_check)
  byte[] bytes = new byte[lump.length];
         ^^^^^
```

Slightly underwhelming, but I also haven't written much code. I'm sure it will come in handy.  
Note I don't ever lint manually, my IDE automatically updates with the latest lints inline. This immediate feedback is why a fast linter is important. No one wants to wait 3+ seconds to start reading what they did wrong.  
I don't pay much mind to those unused vars/params, especially while I'm fleshing things out. I also don't like how it's nagging at me to document _everything_. It doesn't consider an inline comment on a struct member as documentation, it wants a full-fledged javadoc style block comment, for each member. Not happening. I'm going to tweak some of these rules by providing a config file (generated by `dscanner --defaultConfig` and copied into the source tree).  

Let's get into some actual programming.

## VSCode extension

I'm using VSCode, and I'm going to need syntax highlighting, an LSP, maybe a formatter? Looks like `webfreak.code-d` (<https://github.com/Pure-D/code-d>) is the extension for me 🤞  
Upon installing that extension, it complains that `serve-d` (from setting `d.servedPath`) is not installed and prompts me to compile it. Okay, I'll let it do that

```text
> /opt/homebrew/bin/dub build --compiler=ldc2
     Warning Invalid source/import path: /Users/Razish/.dub/packages/dfmt/0.14.1/dfmt/bin
     Warning Invalid source/import path: /Users/Razish/.dub/packages/dscanner/0.11.1/dscanner/bin
     Warning Invalid source/import path: /Users/Razish/.dub/packages/dcd/0.13.6/dcd/bin
     Pre-gen Running commands for dfmt
/bin/sh: rdmd: command not found
Error Command failed with exit code 127: rdmd "/Users/Razish/.dub/packages/dfmt/0.14.1/dfmt/dubhash.d"
Failed to install serve-d (Error code 2)
```

Ugh. That's not nice. This is a few years old too: <https://github.com/Pure-D/code-d/issues/395>  
So we're missing `dcd` and `serve-d`. We can `brew install dcd` and let's try compile `serve-d` from source.

```sh
cd ~/projects
git clone git@github.com:Pure-D/serve-d.git
cd serve-d
dub build
```

...Oh, that was painless. Now we configure VSCode (via `settings.json`) to point to this binary: `"d.servedPath": "/Users/Razish/projects/serve-d/serve-d"`

## What to write?

I recently [wrote an RBSP parser](https://github.com/Razish/bsputil-go) in Go for some practice. That felt a bit clunky. I wonder how D fares for this task?  
RBSP (`.bsp`) a simple binary format used to store level data for games built by Ravensoft on the Quake III engine. The "BSP" comes from Binary Space Partitioning, which describes the tree structure used to quickly traverse (and cull) the level geometry.  
It parses very well in C. It has a header that describes the layout (offset + size) of 18 (depending on the version) "lumps". Think of it as an extremely primitive (uncompressed) zip file for binary data.  
Once the relevant lumps are parsed, I can output a JSONL representation of them to compose pipelines for processing that data: `bsputil foo.bsp entities | jq -r '.classname + " @ " + (.origin // "N/A")'`  
I'm only going to focus on reading some trivial lumps (which are incidentally some of the most practical for the (21 year old 🤘) modding community).

## Reaching for primitive tools

Let's get started by defining that header. I'm sure D has structs, enums etc. Let's see:

```d
enum LumpId
{
  Entities,
  Shaders,
  // ... blah blah blah ...
  NumLumps,
}

struct BspHeader
{
  uint ident;
  uint _version; // for some reason i can't have an identifier called `version`?
  Lump lump[NumLumps];
}
```

That's how the formatter decided to print my code. The opening brace is on a new line ("Allman" style).  
This language is an absolute trash pile, I'm done 🚮  
Thanks for reading.

&nbsp;

---

&nbsp;

Just kidding. It actually gave me a very good suggestion! ...rather, it was an informative _error_.
> Lump lump[NumLumps];  
> source/app.d(39,7): Error: instead of C-style syntax, use D-style `Lump[NumLumps] lump`

I don't hate this so-called "D-style". It's not as egregious as Go-style to me. I believe the "Array of T" detail is very much a part of the type, not the identifier.

- C: `Lump lumps[NumLumps];` "This variable is of `Lump` type, I will refer to it as `lumps` - but actually I have `NumLumps` of them"
- D: `Lump[NumLumps] lump;` "This variable is of `Lump[]` type, I have `NumLumps` of them, and I refer to them as `lumps`"
- Go: `Lumps [NumLumps]Lump` "I will refer to this as `Lumps`, which is a list of `NumLumps` variables of type `Lump`"

After following that suggestion I'm told `undefined identifier NumLumps`, which is actually awesome and exactly what I wanted to hear; enum identifiers must be qualified!  
The correct declaration is `Lump[LumpId.NumLumps] lump;`  
I have always hated the global namespace pollution of enum identifiers in C and C++. At least C++ has "enum classes" as an option to enforce scoping. Tight scoping should just be default behaviour, and avoid prefix stutter where possible (`MyFlags.FlagFoo` -> `MyFlag.Foo`).

How do I know what I've typed is correct? Do enums behave in a sane manner? I hope they start from 0!  
My first instinct for something like this is to make it tell me. I should just print it out. Wait, how do I print a formatted string in D?

```d
  writefln("There are %i lumps", LumpId.NumLumps);
  // or:
  writeln("There are %i lumps".format(LumpId.NumLumps));
  // or:
  writeln(string.format("There are %i lumps", LumpId.NumLumps));
```

😮‍💨 Completely as expected. So far, there is nothing astonishing in how it feels to write D.

```sh
$ dub -q
std.format.FormatException@std/format/internal/write.d(340): incompatible format character for integral argument: %i
# ... stack trace here ...
```

I'm a little disappointed there was no static analysis / compiler warning on the format specifier being wrong. Perhaps I've been spoiled. I also haven't RTFM after all!  
I'm going to assume `%d` is the correct format specifier.

```console
$ dub -q
There are 18 lumps
```

Cool. Nobody has been messing with the universal constants.

## How to structure a project?

What about modules, namespaces, etc?  
Let's do something simple: move my header definition into `parse/header.d` and drive it from `main.d`:

```d
// source/main.d
module main;

import parse.header;
import std.stdio;

void main()
{
  writefln("There are %d lumps", LumpId.NumLumps);
}
```

```d
// source/parse/header.d
module parse.header;

enum LumpId
{
  Entities,
  Shaders,
  // ... blah blah blah ...
  NumLumps,
}

// ... blah blah blah ...
```

Something feels off here, but maybe it's just my javascript speaking.  
Importing a module brings in _everything_ to your top-level namespace, I can simply type `LumpId.Entities` from my `main.d`. How could I scope this further?  
...Future me has come back to answer this; we can use _selective imports_:

```d
import std.stdio : File, writeln;

writeln("this still works");
```

Alternatively, we can use _static imports_, and even rename them to something else:

```d
static import io = std.stdio;

io.writeln("this is even easier than std.stdio.writeln!");
```

I think I'm going to prefer static imports, some renamed imports and a sprinkling of selective imports. I usually like qualifying what I mean. The default behaviour of global pollution feels like the anti-pattern of `using namespace std` in C++ 🤮

By now I've gathered my primitive tools and made sure they're sharp. Let's start getting serious.

## I put on my robe and wizard hat

Let's lay down some goals:

- Parse commandline arguments so we know which file to read
  - This might be trivial, the stdlib has `getopt` after all 🤔 or I can just start off with positional args
- Read the file from disk
- Parse the header
- Parse the desired lump
  - What does memory allocation look like? Should I...not care until I have to?
- Serialise the data as JSONL
  - We need to pick a JSON library - unless it's built in? (spoiler alert: it is 🤩)

That seems simple. Our control flow is as follows:

```d
void main(string[] args)
{
  // parse commandline arguments so we know which file to read
  if (args.length < 2)
  {
    throw new Error("usage: %s <filename> <lump name>".format(args[0]));
  }
  const string inputFilename = args[1];
  const string desiredLump = args[2];

  // read the file from disk
  auto f = File(inputFilename, "rb");

  // parse the header
  BspHeader header = parse.header.read(f);

  // parse the lump(s) and serialise as JSONL
  switch (desiredLump)
  {
  case "entities", "ents":
    auto entities = parse.entities.read(f, &header);
    entities.serialise;
    break;
  case "shaders":
    auto shaders = parse.shaders.read(f, &header);
    shaders.serialise;
    break;
  default:
    throw new UnsupportedLumpException("unsupported lump '%s'".format(desiredLump));
    break;
  }

  f.close;
}
```

This is probably gonna change a bit over time, I'll link the end result at the bottom.

Rather than describing my code as I'm writing it line by line, I'm just going to start writing it off-screen and summarise the best/worst parts of the language as I come across them.

## Some nice language features

### Type aliases

Type aliasing (think `typedef`, `using`) is so intuitive I didn't have to look it up:

```d
alias FooStr = string;
```

### Uniform Function Call Syntax and extending types

What if we wanted to define a function that's directly applicable to an array of `FooStr`, effectively extending two built-in types (`string`, `[]`)?  
UFCS (uniform function call syntax) to the rescue! UFCS essentially means `foo.bar(baz)` is the same as `bar(foo, baz)`. It's syntactic sugar to pass the receiver as the first argument.  
So we can define the following:

```d
void magic(FooStr[] foos)
{
  foos.map!(retro) // retro from std.range, map+each from std.algorithm
    .each!(stdio.writeln);
}

void main(string[] args)
{
  FooStr[] foos = ["hello", "there", "world"]; // this has an implicit cast from string -> FooStr
  foos.magic();
  /* outputs:
    olleh
    ereht
    dlrow
  */

  // we can also call it directly, since `string` can be safely cast to `FooStr`
  ["foo", "bar"].magic();

  // it's also the reason we can call `format` directly on a string literal
  writefln("i have %d apples", 42); // i have 42 apples
}
```

It's a bit like the colon operator for table functions in Lua:

```lua
-- the implicit first argument is the `self`
function Account:withdraw(v)
  self.balance = self.balance - v
end
local acc = Account
acc.withdraw(acc, 100.00) -- this is OK
acc:withdraw(100.00) -- this is the same thing, `:` will pass the receiver as the first argument (i.e. self)
```

This also works for primitive types (bonus example of chaining):

```d
int squared(int v)
{
  return v * v;
}

void main(string[] args)
{
  writeln(16.squared().squared()); // 65536
}
```

### Arrow functions

Another feature where I just wrote some code and _it worked_. The spec calls this a ShortenedFunctionBody.

```d
int squared(int v) => v * v;
```

Seems you can also define these in the middle of a function, but you have to call it explicitly. I can't get the UFCS shortcut to work.

Here's a pretty FP-style usage from the spec:

```d
stdin.byLine(KeepTerminator.yes)
  .map!(a => a.idup)
  .array
  .copy(stdout.lockingTextWriter());
```

### Optional parenthesis

I think this only works when there are no parameters?

```d
auto f = File("foo.bin", "rb");
f.readRaw(myFoo);
f.close; // <-- this is allowed
```

Combine optional parens, arrow functions and UFCS to get the following:

```d
int squared(int v) => v * v;
writefln("IT'S OVER %d", 16.squared.squared);
```

### CTFE - Compile Time Function Execution

Did I mention that all of the above `squared` examples might be executed at _compile_ time?  
This is like C++ `constexpr` on steroids, automatically! Who doesn't love compiler optimisations for free?  
Rule of thumb: if it's a pure function with a static lvalue, it's probably CTFE'd. If it's not pure, executions that don't follow the impure path are still CTFE-able.  
[See more](https://dlang.org/spec/function.html#interpretation)

### Pass by value, \*pointer or &reference? who->cares

There's a bit of churn in C and C++ when dealing with structs passed by value, pointer or reference. Is it `foo.bar` or `foo->bar`?  
It doesn't matter in D. It's always `foo.bar`. It _just knows_ what you want. If you decide to pass by pointer instead of value later on, you don't have to tweak all your references.  
These differences are fairly insignificant in the grand scheme, but the less friction over time, the more stamina and happiness you'll have when writing code. Don't fight a thousand small battles over fine details that should never have mattered. Just. Write. Code. This is the real blessing of D, and I hope to continue experiencing these refreshing breaths.

### foreach loops

These are just nice. Slightly novel syntax, types are inferred for range-based foreach loops. Less clunky than the C++ counterparts.

```d
  for (uint index = 0; index < length; index++) {} // ol' trusty
  foreach (index; 0 .. length) {} // range: numeric
  foreach (element; myArray) {} // range: array iterator
```

### Type inference

Not having to specify the types for your range iterators is nice, but that's not the only place types can be inferred!

```d
const MAX_QPATH = 64u; // no type specified, but it is inferred to be a uint

// _whatever_ File gives us is fine
auto f = File(inputFilename, "rb");

// even function return types!
auto serialise(Item[] items) => items
  .map!(e => e.toJson.toString)
  .join("\n");
```

### Standard Library

This used to be a hot topic in D a few years back. The SparkNotes version is that in the days of D1, there was the standard library called Phobos (aptly named after the largest moon of Mars - oh yeah, D is championed by [Digital Mars](https://www.digitalmars.com/)). The community wanted MOAR so they kinda just added it in the form of complimentary libraries. One of these (Tango) got pretty popular and was affectionately referred to as the second standard library. It's a bit like Boost for C++.  
As of D2, Phobos is a pretty mature library, and the language isn't going through as many breaking changes.  
That typical thing you're trying to do? It's probably in the (Phobos) stdlib.  
That piece of C's stdlib you're so familiar? Oh yeah, you can just reach into that too (`import core.stdc.string : strtok;`)

### Unit Testing

Extremely first-class unit testing!  
It's not uncommon to see unit tests immediately following the implementation:

```d
int foo(string bar)
{
  /* ... */
}

unittest
{
  assert(foo("hello") == 42, "fooing a hello was not a big happy");
}
```

And you can run it like `dub test`, which is roughly equivalent to `dub build -b unittest && ./my_program`.  
Who doesn't love some plain old asserts? No convoluted DSL or 5 new libraries to learn just to write some adequate tests. Now you have an awesome place to put those asserts, and you can _really_ do some TDD right next to the implementation if that's your jam.  
There's also built-in code coverage via `dub -b cov` and `dub -b unittest-cov`, which integrated nicely into VSCode (bonus points for `profile` and `profile-gc` build types too 👌)

Here's a dead simple example of table-based testing I ended up using for entity serialisation:

```d
string serialise(Entity[] entities) => entities
  .map!(e => json.JSONValue(e).toString(json.JSONOptions.doNotEscapeSlashes))
  .join("\n");

unittest
{
  struct TestCase
  {
    Entity[] entities;
    string result;
  }

  TestCase[] testCases = [
    {entities: [], result: ""},
    {entities: [["key": "value"]], result: "{\"key\":\"value\"}"},
    {entities: [["key": "value"], ["key": "value"]], result: "{\"key\":\"value\"}\n{\"key\":\"value\"}"},
  ];
  foreach (testCase; testCases)
  {
    auto serialised = serialise(testCase.entities);
    assert(serialised == testCase.expected,
      "`%s` should serialise correctly, got: `%s`, expected `%s`".format(
        testCase.entities, serialised, testCase.expected));
  }
}

// after intentionally breaking one of the cases:
// core.exception.AssertError@source/parse/entities.d(137): `[["keay":"value"]]` should serialise correctly, got: `{"keay":"value"}`, expected `{"key":"value"}`
```

### Breaking out of multiple loops

Pretty simple, but nice to have when you need it.

```d
outer: for (size_t i = 0; i < input.length; i++)
{
  // ... doing some stuff ...

  while (looksGood(i))
  {
    if (input[i] == theBadThing)
    {
      break outer;
    }
  }
}

```

### Voldemort Types

This gets an honourable mention just for the name 😆

```d
auto createDarkLord() {
  struct Voldemort {
    string whoAmI() const {
      return "He who must not be named";
    }
  }

  return Voldemort();
}

const auto youKnowWho = createDarkLord();
// youKnowWho is He who must not be named of type const(main.createDarkLord().Voldemort)
stdio.writefln("youKnowWho is %s of type %s", youKnowWho.whoAmI(), typeid(youKnowWho));
```

There's actually no way to refer to the `Voldemort` type, we can only use type inference from the factory function.

## Some un-nice language features

### `string` is quite literally a string of bytes

Strings are a bit cooked. It seems this is the case in a lot of modern languages that hide the nastiness of C (or pascal) strings behind a facade and try to present this nicely abstracted concept of "a range of text", but still try to reap the benefits of being "low level". Go is no exception here.  
The file format I'm parsing has an array of structs stored _in binary_. A string of 64 chars followed by 2 integers per element. Strong emphasis on "_in binary_".  
This means the string `"foo"` is 3 characters (`[ 'f', 'o', 'o' ]`) followed by 61 null bytes. If I parse that out into a struct with `File.readRaw`, then assign it to my nicely abstracted `string` variable like `myStr = myBytes.idup`, I end up with: `foo\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000\u0000`.  
...but of course, you have to call `.fromStringz` to parse a null-terminated string from an array of bytes. _Obviously!_ Also, the `.idup` is there to make an immutable copy of the bytes. You'll see a lot of `idup` thrown around the place.

It's no better in Go. We don't have a built-in `.fromStringz` there, we have to do something ridiculous like this ourselves:

```go
func CToGoString(b []byte) string {
  i := bytes.IndexByte(b, 0) // find the first null byte
  if i < 0 {
    i = len(b)
  }
  return string(b[:i])
}

myBytes := []byte("foo\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00")
fmt.Println("c-string len: ", len(CToGoString(myBytes[:]))) // 3
fmt.Println("gostring len: ", len(string(myBytes[:]))) // 15
```

### Dynamic array appending is strange

Thought you could write `myArray.append(42);` or `myArray += 42`? Nah. Actually you want to do `myArray ~= 42;`. Wat?  
The docs weren't 100% clear on this either, I didn't trust it until I tried it.

### You can't name a function the same as the top-level module it's in

...if you're importing another module from the same top-level module.

```d
// parse/header.d
module parse.header;

// ...
```

```d
// parse/entities.d
module parse.entities;

import parse.header; // <-- this is the catalyst for the following error

alias Entity = string[string];
alias EntityString = byte[];

static Entity[] parse(EntityString bytes) // function `parse.entities.parse` conflicts with import `parse.entities.parse` at source/parse/entities.d(8,8)
{
  // ...
}

static Entity[] parseEntities(EntityString bytes) // this works fine, but introduces stutter
{
  // ...
}
```

### std.json default options

Pop quiz. How would you serialise a(n object containing a) string as JSON?  
Perhaps you'd write the following:

```d
auto foo = "/path/to/foo";
stdio.writeln(foo.JSONValue.toString); // surely this prints `"/path/to/foo"`?
```

But you'd be wrong. I mean it would work, and even be spec compliant. But you'd get `"\/path\/to\/foo"` instead.  
What you _actually_ intended was:

```d
stdio.writeln(foo.JSONValue.toString(JSONOptions.doNotEscapeSlashes));
```

Some absolute madman decided the _default_ for serialising a string as JSON should _escape the quotes_. Whyyyyyyyy 😩  
Okay it's not the end of the world...but...whyyyyyy! We're _probably not_ injecting this straight into an XML body in the majority of cases. Even the de facto reference implementation in JS doesn't _default_ to escaping quotes.

## End Result

You can find the result of my little hackathon on my Github: [Razish/bsputil-d](https://github.com/Razish/bsputil-d).  
I highly recommend having a read through the code to see how I've managed to do things.  
If you want to play around with it, you can source a community-made BSP file from [this section](https://jkhub.org/files/category/13-free-for-all/) - unzip the downloaded PK3 file.

## Final thoughts

Wow. That was an interesting experience. I mostly really enjoyed writing D. Sure there are some slightly painful things, but nothing that was truly inherent to D or that couldn't be overcome with the slightest bit of perseverance. Writing and debugging a somewhat flexible tokeniser/scanner was something I hadn't done in a while, and I managed to pull it off somewhat neatly using slices of an immutable buffer in a new (to me) language. I wouldn't have minded having something like Go's test.Scanner handy though.  
It was fairly easy to intuit how I should be structuring things, naming things, etc and the tools are very helpful to guide you along the path if you stray too much. I didn't feel like I was _fighting against_ the tooling.

There's a lot to the D landscape that I haven't explored yet. I'm keen to go through the DLang tour next and learn more about this language.  
Overall, I honestly felt more comfortable writing D than writing Go, and it felt equally (if not more) powerful, expressive, handy and featureful. I think there's a lot of room for D to shine in areas like tools programming. I haven't personally assessed the performance, but it seems completely adequate for anything but HPC. You can also run in a [non-GC mode](https://dlang.org/blog/2017/06/16/life-in-the-fast-lane/) if you're concerned about that.  
I'm very keen to explore the C(++) interop more, since I still work a lot with C(++) code and it's not that big a leap.

I said I wanted to evaluate "how easy is this to just pick up and hit the ground running" - I think it was very easy. I was able to guess most of the syntax, features, structure. The tooling (compiler:ldc, linter:dscanner, build tool:dub) told me a lot with very good errors/warnings (much less convoluted than C++, much more detailed than JS). Everything else was immediately discoverable by searching `dlang the thing` and [R-ing The FM](https://dlang.org/phobos/index.html).  
It might not be good as someone's _first_ language to learn, but anyone looking to pick up a second or third language (perhaps one that is low(er) level) and is even slightly put off by C++ should absolutely give D a shot.

## Future Work

Now that I've given it an honest go as a complete novice, I want to spent some more time going through the DLang tour and learn the more magical parts of the language. I'm sure I've skipped over some incredibly useful features - which is _amazing_, because I was still able to be productive and write correct code with minimal friction. It's exciting to think how much better it could be after spending the time to really learn the language. I think I might give templates a real chance - as someone who mostly avoids C++ template hell, too much STL or ever using Boost.

I might pick up another modern-ish language and run it through a similar gauntlet. Probably a different task, I'm already sick of this one 😆  
I'm thinking something like Nim, Crystal, Rust, Zig or Racket (lol). Guess I'll just roll a dice!

Stay tuned for more insane rants where I do bad things incorrectly maybe? 😇
