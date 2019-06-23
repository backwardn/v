# The V Programming Language

https://vlang.io

Documentation: https://vlang.io/docs

Twitter: https://twitter.com/v_language


## Code Structure

I tried making the code of the compiler and vlib as simple and readable as possible. One of V's goals is to be open to developers with different levels of experience in compiler development. Compilers don't need to be black boxes full of magic that only few people understand.

The compiler itself is located in `compiler/`

It has only 8 files (soon to be 7):

1. `main.v` The entry point. 
- V figures out the build mode.
- Constructs the compiler object (`struct V`).
- Creates a list of .v files that need to be parsed.
- Creates a parser object for each file and runs `parse()` on them (this should work concurrently in the future). The parser emits C or x64 code directly. For performance reasons, there are no intermediate steps (no AST or Assembly code generation).
- If the parsing is successful, a single C file is generated by merging the output from the parsers and carefully arranging all definitions (C is a single pass language).
- Finally, a C compiler is called to compile this C file and generate an executable or a library.

2. `parser.v` The core of the compiler. This is the largest file (~3.5k loc). `parse()` method asks the scanner to generate a list of tokens for the file it needs to parse. Then it simply goes through all the tokens one by one.

   In V objects can be used before declaration, so there are 2 passes. During the first pass it only looks at declarations and skips function bodies. It memorizes all function signatures, types, consts, etc. During the second pass it looks at function bodies and generates C  (e.g. `cgen('if ($expr) {'`) or machine code (e.g. `gen.mov(EDI, 1)`).

   The formatter is embedded in the parser. Correctly formatted tokens are emitted as they are parsed. This allowed to simplify the compiler and avoid duplication, but slowed it down a bit. In the future this will be fixed with build flags and separate binaries for C generation, machine code generation, and formatting. This way there will be no unnecessary branching and function calls.
   
3. `scanner.v` The scanner's job is to parse a list of characters and convert them to tokens. It also takes care of string interpolation, which is a mess at the moment.

4. `token.v` This is simply a list of all tokens, their string values, and a couple of helper functions.

5. `table.v` V creates one table object that is shared by all parsers. It contains all types, consts, and functions, as well as several helpers to search for objects by name, register new objects, modify types' fields, etc.

6. `cgen.v` The small `Cgen` struct helps generate C code. It's also shared by all parsers. It has a couple of functions that allow to go back and set something that was previously unknown (like with `a := 0` => `int a = 0;`). Some of these functions are hacky and need improvements and simplifications.

7. `fn.v` Handles declaring and calling normal and async functions and methods. This file is about 1000 lines of code, and has some complex logic. It needs to be cleaned up and simplified a bit.

8. `json.v` defines the json code generation. This file will be removed once V supports comptime code generation, and it will be possible to do this using the language's tools.

9. `x64/` is the directory with all the machine code generation logic. It will be available in early July. Obviously this is the most complex part of the compiler. It defines a set of functions that translate assembly instructions to machine code, it builds complicated binaries from scratch byte by byte. It manually builds all headers, segments, sections, symtable, relocations, etc. Right now it only has basic support of the x64 platform/Mach-O format, and it can only generate `.o` files, which then have to be linked with `lld`. 

The rest of the directories are vlib modules: `builtin/` (strings, arrays, maps), `time/`, `os/`, etc. Their documentation is pretty clear.


## Installing V from source

### Linux and macOS

```bash
mkdir -p ~/code && cd ~/code  # ~/code directory has to be used (it's a temporary limitation)
git clone https://github.com/vlang/v
cd v/compiler
wget https://vlang.io/v.c # Download the V compiler's source translated to C
cc -w -o vc v.c           # Build it with Clang or GCC
./vc -o v .               # Use the resulting V binary to build V from V source
./v -o v .                # Bootstrap the compiler to make sure it works
```

That's it! Now you have a V executable at `~/code/v/compiler/v`.

You can create a symlink so that it's globally available:

```
sudo ln -s $HOME/code/v/compiler/v /usr/local/bin/v
```

### Windows

V works great on Windows Subsystem for Linux. The instructions are the same as above.

If you want to build v.exe on Windows without WSL, you will need Visual Studio. Microsoft doesn't make it easy for developers.  Mingw-w64 could suffice, but if you plan to develop UI and graphical apps, VS is your only option.

V temporarily can't be compiled with Visual Studio. This will be fixed asap.

### Testing

```
$ v

V 0.0.12
Use Ctrl-D to exit

>>> println('hello world')
hello world
>>>
```

Now if you want, you can start tinkering with the compiler. If you introduce a breaking change and rebuild V, you will no longer be able to use V to build itself. So it's a good idea to make a backup copy of a working compiler executable.


### Running the examples

```
v hello_world.v && ./hello_world # or simply
v run hello_world.v              # This builds the program and runs it right away

v word_counter.v && ./word_counter cinderella.txt
v news_fetcher.v && ./news_fetcher
v tetris.v && ./tetris
```

In order to build Tetris and anything else using the graphics module, you will need to install glfw and freetype.

If you plan to use the http package, you also need to install libcurl.

glfw and libcurl dependencies will be removed soon.

```
Ubuntu:
sudo apt install glfw libglfw3-dev libfreetype6-dev libcurl3-dev

macOS:
brew install glfw freetype curl
```
