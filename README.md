# Journal
## Feb 9
First meeting. Created VM with more memory (issue with previous VMs was RAM, not storage after all). It actually worked this time: I compiled and ran the Python interpreter. 

Forked Python to create my own repo. Installed VSCode extension for `.gram` files.  

## Feb 12
Changed `if` to `se` in `python.gram`. Ran configure script and recompiled. Didn't work: code with `se` raised syntax error and code with `if` worked fine. 

Meanwhile, how should I translate keywords like `return`? Issue with translating keywords like this one to a romance language: grammatical mood. Should keywords like `return` be in the indicative mood (e.g. "retorna"), imperative mood ("retorne"), or infinitive mood ("retornar")? Keywords should be in the same grammatical mood as functions for consistency and readability, so I can just ask my friends in Brazil who are now in the industry what grammatical mood they use for function names. Answer: indicative seems to be more popular.

## Feb 16
Second meeting. `se` issue was that I hadn't been rerunning the parser generator (which uses the `.gram` file to generate the parser). New issue: syntax error in the standard library. I should've seen this coming: the Python standard library is in fact written in Python, not pybr.

Looked into compiling Python without standard library. Impossible, unless I either:
1. Transpile the whole Python source of the standard library to pybr. This would avoid syntax errors in the standard library because the source would now be written in legal pybr. 
2. Compile the whole Python source of the standard library (everything in `Lib/`) to `.pyc` using unmodified Python. This would avoid syntax errors in the standard library because it would no longer consist of source code, but of precompiled code (which is the same for Python and pybr).

Both fixes above sound intractable (especially the first one), and probably aren't long-term solutions since the standard library changes *very often*. So new plan: instead of turning the Python interpreter into a pybr interpreter, internally convert pybr source to Python AST and execute the AST (either with an internal custom Python install or with user's install; both approaches have pros and cons). One big plus side of this new plan is that pbybr–Python interop should be trivial once I get pybr to work.

Looked into `Python/compile.c`. Not useful: it compiles AST, but what I'm worried about is getting to the AST. 

Function `parse` in `Lib/ast.py` generates Python AST from Python source. You can't specify a grammar, but maybe I can go to the function's source and change the grammar it uses for my own? No: `parse` is just a wrapper for the `compile` function, which is builtin. So I'll have to either write my own parser or somehow reuse the Python parser. The second one is easier, so let's try that.

Current plan: change the grammar to generate a pybr parser, but just use the parser to compile pybr source to Python AST. Run the AST using the unmodified Python interpreter.

## Feb 18
`Python/ast.c` is where the magic happens. The main function is `PyAST_FromNodeObject` (and its wrapper, `PyAST_FromNode`, which is probably going to be more useful). Its input is the root of a CST, which is no problemo: Python and pybr have the same CST grammar. Its output is a `mod_ty` (a module). The book is kinda vague about what exactly is a `mod_ty`. Since `PyAST_FromNode` outputs a `mod_ty`, I conclude that it encodes Python source in AST form.

I'll put `ast.c` aside for a sec and instead try to understand the AST API from where it's used, not where it's written. `pythonrun.c` is the top-level execution of Python code. So understanding how `pythonrun.c` uses the AST API may be my best shot at understanding how to do so myself. In the function `run_mod`, `pythonrun.c` uses a function called `_PyAST_Compile` to generate a `PyCodeObject`, which is then fed into `run_eval_code_obj`. But does `_PyAST_Compile` compile *to* AST or *from* AST? (Note from future: the answer is *from*, read on for my reasoning.)

The function `pyrun_file` seems to have the answer—three answers, in fact. It returns a `PyObject` (not to be confused with a `PyCodeObject`), but what's promising is that its input is (finally) *not* some internal struct, but a straight-up file. I won't worry about what `pyrun_file` returns for now: what I'm interested in is what's going on inside. First, `pyrun_file` passes that file to a function called `_PyParser_ASTFromFile`, putting the result into a `mod_ty`. (This confirms that `mod_ty` indeed encodes Python AST.) Then `pyrun_file` calls `run_mod` on that `mod_ty`. (So the second revelation is that this gives us a way to execute a `mod_ty`, sweet.) And now for the third revelation: looking at the body of `run_mod`, I see the answer to whether `_PyAST_Compile` compiles *to* AST or *from* AST. The answer is that it compiles *from* AST: this is because `run_mod` takes a `mod_ty` (which I now know is AST) and passes it to `_PyAST_Compile` to generate a `PyCodeObject`. So I may not have to worry about the `PyCodeObject` type (which is compiled AST), the `_PyAST_Compile` function (which compiles AST to `PyCodeObject`), or the `run_eval_code_obj` function (which runs a `PyCodeObject`).

Conclusion: the actually useful discoveries were `_PyParser_ASTFromFile` (which gives an AST) and `run_mod` (which runs an AST). So my plan is to:
1. Edit the Python grammar into a pybr grammar.
2. Regenerate the parser, which should give me a new `_PyParser_ASTFromFile`.
3. Use it to transform pybr source into Python AST.
4. Feed the AST into the run_mod of an intact Python interpreter (including the original Python parser, not the one I generated in step 2) to run the AST I generated in step 3. Crucially, functions in the standard library will be parsed with the original Python parser, so no more syntax errors.

Cool.

## Feb 22
Instead of using `_PyParser_ASTFromFile` to go from file to AST, I can use `PyParser_ParseFileObject` to go from file to CST and then use `PyAST_FromNodeObject` to go from CST to AST. The two approaches are probably equivalent. But the fewer functions I use, the less likely it is that I use them incorrectly, so I’ll probably just use `_PyParser_ASTFromFile` after all. Also, I found where this function is defined: it's in `Parser/peg_api.c`, which also contains a `_PyParser_ASTFromString` function that should be useful for testing.

Finally, looking at how modules are defined in CPython I learned that C has union types. This is either cool or horrifying; I'm leaning toward cool.

## Feb 23
Third meeting. CS 81c confirmed.

Created `pybr-parser-playground` branch with a new file, `playground.c`, where I'll play around with the parser internals (particularly the functions I found on Feb 18) to understand how to use them. Moved this journal to GitHub. Didn't like GitHub Pages, so I'll maintain the journal as the README of its own repo.

## Feb 27
I found the header file where the `mod_ty` type is defined, as well as the one where the `_PyParser_ASTFromString` function (and `_PyParser_ASTFromFile`, but that's later) is defined. So I'm able to use these symbols in `playground.c`. But `run_mod` isn't in any header files (I checked). I'm able to use it in `playground.c` if I `#include pythonrun.c`, but my understanding is that including `.c` files is a bad idea. So if there are symbols from the parser internals that I want to use in `playground.h`, but these symbols aren't in any header files, I guess I just... add them to a new header file? Or I can also just copy-and-paste the symbols I want into `playground.c`. Both feel hacky.

I'll worry about this later; having `_PyParser_ASTFromString` means that I *should* be able finally compile pybr source to AST, after I change and recompile the parser. And, yknow, figure out what exactly are the five `_PyParser_ASTFromString` parameters:

1. `char* str`. This one is easy: it's just the Python source to compile.
2. `PyObject* filename`. I'll just pass some dummy filename to something that gives me a `PyObject*` or whatever. Easy, right? Not so fast. Almost every usage of a `PyObject` encoding a filename that I found gets the `PyObject` from somewhere else. I did find one instance of *creating* a `PyObject` from a filename string, but it used some absolutely *wacky* C macro nonsense. Specifically, the ritual involves the `_Py_DECLARE_STR` and `_Py_STR` macros from `pycore_global_strings.h`. So I tried to replicate the macro nonsense in `playground.c`, but couldn't get it to not give me a syntax error. Then I found a different potential approach: there's a function called `PyUnicode_FromWideChar` in `unicodeobject.c` that takes a `wchar_t*` and spits out a `PyObject*`. I presume that `wchar_t*` is a Unicode string, given the type's name and the name of the function I found it in. So it shouldn't be hard to convert a `char*` filename to a `wchar_t*` filename, then to use `PyUnicode_FromWideChar` to convert *that* to a `PyObject*` filename. Or, better yet, there's probably a function somewhere that takes me directly from `char*` to `PyObject*`. If I find it, I can skip the useless Unicode conversion. In any case, I'll get back to this `PyObject* filename` parameter later.
3. `int mode`. I looked around and found the compilation modes in `compile.h`. One of them is called `Py_file_input`; I'm technically compiling from a string (for now), not a file, but this compilation mode  makes the most sense (and everything so far suggests that compiling from a string delegates to compiling from a dummy file anyway). So I'll use `Py_file_input` as the compilation mode. And the only usage of `_PyParser_ASTFromString` that doesn't get the `mode` from somewhere else passes `Py_file_input` as the `mode`, so I'm pretty sure that's right.
4. `PyCompilerFlags* flags`. They're covered in Chapter 8.4 of the book, so I'll look at that later.
5. `PyArena* arena`. Same as above, but in Chapter 10.8. I'll also check out the `pyarena.c` and `pyarena.h` files, as well as the `_PyArena_New` function.

## Feb 28
The flags chapter was kinda useless. It doesn't list any compiler flags (it only lists "future flags", which I currently don't care about), much less what any of the compiler flags do. The chapter also doesn't say how flags are represented/used internally nor give any functions or files related to compiler flags. On the other hand, the code has more answers—though not all of them.

The compiler flags are all in `compile.h`. Flags are implemented as a bit set: there's a constant (also defined in `compile.h`) called `_PyCompilerFlags_INIT` representing an empty set of flags, and you add more flags through bitwise disjunction. This bit set, plus the Python version, are the only two members of the `PyCompilerFlags` struct.

For now, I just want to compile a string of simple source code and not worry about the details. For example, right now I couldn't care less if the compiler is optimizing out `assert` statements, or allowing top-level `await`, or whatever else. So, in theory, I'd just use `_PyCompilerFlags_INIT` and call it a day. In practice, there are two flags which I'm not sure I can safely ignore.

One of them is called `PyCF_IGNORE_COOKIE`. I don't know what it is; the code, the book, and the web have no answers. Why do I care about it? Here's the situation. There are two functions that run a string of Python source: `pymain_run_command` (in `Modules/main.c`) and `PyRun_SimpleStringFlags` (in `Python/pythonrun.c`). The former is a wrapper for the latter, the difference being that while `pymain_run_command` only has one parameter (the source code), `PyRun_SimpleStringFlags` also takes a set of flags. This is great news: it means that, by looking at what flags `pymain_run_command` passes to `PyRun_SimpleStringFlags`, I'll know what flags I should use if I want to run a string of Python source. And again, *in theory*, I'd expect the empty set `_PyCompilerFlags_INIT` to be enough. Instead, the `pymain_run_command` does a `|=` with the `PyCF_IGNORE_COOKIE` flag. So, should *I* add this flag too? I'm leaning toward "it probably doesn't matter".

The other potentially important flag is called `PyCF_ONLY_AST`. I did [find something about this on the web](https://docs.python.org/3/library/ast.html#ast-compiler-flags), though in a completely different context. The `compile` function built into Python also takes a set of flags, and one of these flags is called `PyCF_ONLY_AST`. `compile` returns an AST object if this flag is present, and a code object otherwise. But this is in Python land; so what does this mean in C land? After a bit of looking around, I found where several built-in Python functions (including `compile`) are implemented in C: `Python/bltinmodule.c`. The C function backing the `compile` Python function is `builtin_compile_impl`. Just like its Python counterpart, `builtin_compile_impl` takes a set of flags, and returns an AST if the `PyCF_ONLY_AST` flag is present (we're now in C land, so I'm talking about the actual `PyCF_ONLY_AST` flag defined in `compile.h`) and returns a code object otherwise. `compile` being able to return completely different things made perfect sense in the dynamically-typed lands of Python, but how is this done in C, where an AST is `mod_ty` and a code object is `PyCodeObject`? (No, it's not union types.) The answer is in `Py_CompileStringObject`, a function in `pythonrun.c` (an old friend) called by `builtin_compile_impl`. First, `Py_CompileStringObject` compiles the string to a `mod_ty`. If `PyCF_ONLY_AST` is present, the `mod_ty` is converted to a `PyObject` (using a function called `PyAST_mod2obj`, which I may find useful by itself) and returned. But if `PyCF_ONLY_AST` is instead absent, that `mod_ty` is then further processed into a `PyCodeObject*` via `_PyAST_Compile` (another old friend) and "converted" to a `PyObject*` by simply casting the pointer.

Ok, so what was the point of all that? Well, I learned two things:

1. First, I have the answer I was originally looking for: should I include `PyCF_ONLY_AST` in the flags I pass to `_PyParser_ASTFromString`? The answer is a definitive *it doesn't matter*. This flag is used by functions that can either parse to AST or compile to Python bytecode (like `builtin_compile_impl` and `Py_CompileStringObject`), so this flag is almost certainly not checked by functions like `_PyParser_ASTFromString` which always parse to AST.
2. But the second and more important takeaway is that, by going down this rabbit hole, I now have a way of using the Python function `compile` from within C, which is probably a much easier way of compiling source to AST than my current approach (using `_PyParser_ASTFromString`). If I do this, the answer to whether I should include the `PyCF_ONLY_AST` flag is a definitive *yes*.

## Mar 3
Fourth meeting. Prof. Vanier suggested that the `PyCF_IGNORE_COOKIE` flag tells the compiler not to check ``__pycache__`` for a previously-compiled version of the file being compiled. This agrees with the usages of this flag found on Feb 28. (It's also literally the name of the flag; why didn't *I* think of that?) In any case, this means that it probably doesn't matter if I use this flag or not, but just in case I'll use it while I'm compiling strings instead of files.

I've been diving straight into the implementation details, but I actually hadn't yet stopped to take a few steps back and think about one simple fact: I need two parsers, a Python parser and a pybr parser (which is either a hijacked Python parser or a parser I write from scratch, probably/hopefully the former). I mean, I knew that I'll need both parsers before this "take a few steps back" session (see "current plan" in the Feb 16 and Feb 18 entries), I just hadn't really thought about it. Like, what is the _precise_ reason why I need two parsers? And what exactly does that entail? And how would that work?

First, the "why". I wrote that it's because I get a "syntax error in the standard library", which itself is because "the Python standard library is in fact written in Python, not pybr". But while it's true that _some_ of the standard library is written in Python, this still doesn't answer the question of _why_ any Python code (in the standard library or otherwise) is being parsed at all. If I fire up a Python REPL and run

```python
import ast
print(ast.dump(compile("print('hello')", "dummy.py", "exec", ast.PyCF_ONLY_AST), indent=True))
```

I get

```python
Module(
 body=[
  Expr(
   value=Call(
    func=Name(id='print', ctx=Load()),
    args=[
     Constant(value='hello')],
    keywords=[]))],
 type_ignores=[])
```

As expected, calling a standard library function just adds a `Call` node to the AST; the function's body is not parsed (at the parsing stage, there's no reason to). This means that, if nothing else, it should be possible to parse source code to AST even after changing the grammar to pybr.

If the standard library function being used in the example were, say, the `load` function from the `json` module, parsing to AST is as far as we'd be able to go because `json.load` is written in Python. In particular, it wouldn't be possible to compile the resulting AST to bytecode since that _would_ require parsing the body of `json.load`, which would be written in a now-illagal grammar. But the `print` function is _not_ written in Python, so the AST above (with the `print` call) _can_ be compiled to bytecode without parsing any Python. Matter of fact, in theory nothing's stopping us running it as well.

And indeed, I got a syntax error before I even tried to parse anything: the error happens while `make` is running a program called `Programs/_freeze_module`. Also in `Programs`, there's a `_freeze_module.py` and a `_freeze_module.c`, both with the same functions and parameters (and, according to the documentation, both compile to roughly the same bytecode). So which of the two is the source of the `_freeze_module` executable where the error happens? My initial theory was that it's the Python version; after all, I'm getting a Python syntax error _and_ I'm only getting it on the CPython clone with the modified Python grammar. But then I asked myself: how is a Python script being executed while the Python interpreter is still being compiled? Maybe `_freeze_module.py` is being run with the pre-existing Python installation on my machine, and if no such installation exists, `_freeze_module.c` is chosen instead. No, if that were the case then I wouldn't be getting a syntax error. So I decided to test the theory that, despite the Python syntax error (which, again, only happens if I change the Python grammar), the source is actually the `.c` version. I deleted `_freeze_module` and `_freeze_module.py`, then ran `make` again. Sure enough, `_freeze_module` compiled just fine and I got the syntax error. Then I tried deleting `_freeze_module` and `_freeze_module.c` instead, and this time `make` failed. So the Python syntax error is somehow happening in `_freeze_module.c`. 

Looking at `_freeze_module.c`, I now see why. It calls `Py_CompileStringExFlags`, a wrapper for our old friend `Py_CompileStringObject`, the promising function from the previous entry (Feb 28). In fact, `Py_CompileStringExFlags` or _its_ wrapper, `Py_CompileStringFlags`, may end up being more helpful than `Py_CompileStringObject` since they deal with the filename issue I was struggling with on Feb 27. But I digress. The cause of the syntax error is that `_freeze_module.c` is compiling the standard library for "freezing", whatever that is. 

I _could_ change the `Makefile` so that, instead of using `_freeze_module.c`, it runs `_freeze_module.py` with the Python installation on my machine. That would use the builtin `compile` function of an unmodified Python interpreter, and would therefore compile (and "freeze") the standard library of the modified CPython clone (i.e. with the pybr grammar) with no syntax errors. I may not want to assume that the end users of pybr have their own Python installation, but they're not the ones parsing the standard library: _I_ am. The end users get the interpreter + library already compiled/frozen/whatever. Note that I won't actually do this because there still _is_ a good reason to have a Python parser within pybr (read on), so I might as well keep using `_freeze_module.c`.

To recap:
1. Parsing, compiling, and running pybr wouldn't use a Python parser. 
2. Python scripts in the build process are not the problem. If any are actually used, they wouldn't use the Python parser within pybr anyway.
3. What's currently giving me a syntax error is a "freeze" step that does require a Python parser. But freezing the standard library is a one-and-done step that won't happen at the end user, and the Python parser is requires can be anywhere. So the Python parser doesn't have to be within pybr, and even if it is, the Python parser doesn't have to be shipped to the end user.

So why do I need a Python parser within pybr? One reason is to use the parts of the Python standard library that are written in Python. And even if I choose to translate them to pybr (which I probably won't), there's also the matter of integrating with third-party libraries.

That's the _why_. The next two rabbit holes to descend are _what_ exactly having two parsers entails, and _how_ to make this work.

## Mar 8
I have an idea for how to answer the _what_ question: a controlled experiment. I will:

1. Make two fresh clones of CPython.
2. Change the grammar in one of them to the grammar with `se` instead of `if`. The other clone is a control group.
3. Generate both parsers, compare them at each step.

This should tell me what, exactly, are the differences between a Python parser and a pybr parser. For example, is the difference just one file, or fifty different files scattered across multiple directories?

```zsh
git clone https://github.com/python/cpython.git
cp -r cpython cpython-pybr
diff -arq cpython cpython-pybr
```
As expected, `diff` outputs nothing. For now.

```zsh
cd ~/Developer/pybr
git checkout pybr-change-grammar
cp Grammar/python.gram ~/Desktop/cpython-pybr/Grammar/python.gram
cd ~/Desktop
vim -d cpython/Grammar/python.gram cpython-pybr/Grammar/python.gram
```
(That last `vim -d` line is a nice visual sanity check.) 

Then I ran `./configure` on both `cpython` and `cpython-pybr` and compared them again with `diff -arq cpython cpython-pybr`. In addition to the differences in `Grammar/python.gram` from the previous step, running `./configure` created a few _new_ differences. But `vim -d`ing the differing files showed that all the new differences from `./configure` had to do with the fact that the directory names were different (`cpython-pybr` versus `cpython`), not with the changed grammar. Perhaps a _more_ controlled experiment would've been to give the directories identical names (and to not put both in the same parent directory, in order to make this possible). Oh well.

Actually, let me go ahead and do that.

(...)

There. The two directories are now identical save for `Grammar/python.gram`.

Then, in the directory with the pybr grammar, I ran `make regen-pegen`. Another `diff`, and a new difference is revealed: the pybr version and the control group have different `Parser/parser.c`s. The differences between the two are exactly what you'd expect: every instance of `if` is now `se`. There's also a `__pycache__` in the `Tools/peg_generator/pegen` of the pybr version, which means that some Python code was run (presumably with no syntax errors, otherwise I would've seen them) even though I just messed up the grammar. This is no surprise: the Python script for regenerating the parser is, of course, parsed with the original Python parser. What's interesting about this is: how is a Python script being run if I haven't yet run `make` to finish compiling the Python interpreter? Like I theorized in the Mar 3 entry, it may be that the script is being run with the Python installation on my machine. This would be strange, though, since Python can be compiled on platforms without a builtin Python installation.
It's curious, but I'm kind of in the middle of something.

Moving on, I ran `make regen-ast` in the pybr version. The only difference is that now there's also a `__pycache__` in `Parser`.

The next step would be to compare the two clones after running `make` to see how else, besides `parser.c`, the two clones differ. The problem is, the pybr version will crash while `make`ing because of the freezing stuff. Tomorrow, I'll try this out anyway.

## Mar 9
Fifth meeting. Prof. Vanier explained what is "freezing" and sent me a link for further reading. It's actually pretty cool, I'm surprised I had never head of it. Though I personally believe that, if you need to write a cross-platform application and you can't assume that the user has Python installed, and your solution is "write it in Python anyway and just Gorilla-Glue the interpreter to the executable", then you need help. Then again, we live in a world where grown adults use Electron.js.

To wrap up the controlled experiment, I ran `make` on both CPython clones and checked them for differences. Unsurprisingly, the one that crashed halfway was missing several generated scripts, frozen modules, and ``__pycache__``s that were present in the control group. But I didn't see any other differences. So it seems that the difference a Python interpreter and a pybr interpreter really is just `parser.c`. This is a welcome answer to the _what_ question.
