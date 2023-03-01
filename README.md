# Journal
## Feb 9
First meeting. Created VM with more memory (issue with previous VMs was RAM, not storage after all). Actually worked this time: compiled and ran Python interpreter. 

Forked Python to create my own repo. Installed VSCode extension for `.gram` files.  

## Feb 12
Changed `if` to `se` in `python.gram`. Ran configure script and recompiled. Didn't work: code with `se` raised syntax error and code with `if` worked fine. 

Meanwhile, how should I translate keywords like `return`? Issue with translating keywords like this one to a romance language: grammatical mood. Should they be in the indicative mood (e.g. "retorna"), imperative mood ("retorne"), or infinitive mood ("retornar")? Keywords should be in the same grammatical mood as functions for consistency and readability, so I can just ask my friends in Brazil who are now in the industry what grammatical mood they use for function names. Answer: indicative seems to be more popular.

## Feb 16
Second meeting. `se` issue was that I hadn't been rerunning the parser generator (which uses the `.gram` file to generate the parser). New issue: syntax error in the standard library. I should've seen this coming: the Python standard library is in fact written written in Python, not pybr.

Looked into compiling Python without standard library. Impossible, unless I either:
1. Transpile the whole Python source of the standard library to pybr. This would avoid syntax errors in the standard library because the source would now be written in legal pybr. 
2. Compile the whole Python source of the standard library (everything in `Lib/`) to `.pyc` using unmodified Python. This would avoid syntax errors in the standard library because it would no longer consist of source code, but of precompiled code (which is the same for Python and pybr).

Both fixes above sound intractable (especially the first one), and probably aren't long-term solutions since the standard library changes *very often*. So new plan: instead of turning the Python interpreter into a pybr interpreter, internally convert pybr source to Python AST and execute the AST (either with an internal custom Python install or with user's install; both approaches have pros and cons). One big plus side of this new plan is that pbybr–Python interop should be trivial once I get pybr to work.

Looked into `Python/compile.c`. Not useful: compiles AST, what I'm worried about is getting to the AST. 

Function `parse` in `Lib/ast.py` generates Python AST from Python source. You can't specify a grammar, but maybe I can go to the function's source and change the grammar it uses for my own? No: `parse` is just a wrapper for the `compile` function, which is builtin. So I'll have to either write my own parser or somehow reuse the Python parser. The second one is easier, so let's try that.

Current plan: change the grammar to generate a pybr parser, but just use the parser to compile pybr source to Python AST. Run the AST using the unmodified Python interpreter.

## Feb 18
`Python/ast.c` is where the magic happens. The main function is `PyAST_FromNodeObject` (and its wrapper, `PyAST_FromNode`, which is probably going to be more useful). Its input is the root of a CST, which is no problemo: Python and pybr have the same CST grammar. Its output is a `mod_ty` (a module). The book is kinda vague abt what exactly is a `mod_ty`. Since `PyAST_FromNode` outputs a `mod_ty`, I conclude that it encodes Python source in AST form.

I'll put `ast.c` aside for a sec and instead try to understand the AST API from where it's used, not where it's written. `pythonrun.c` is the top-level execution of Python code. So understanding how `pythonrun.c` uses the AST API may be my best shot at understanding how to do so myself. In the function `run_mod`, `pythonrun.c` uses a function called `_PyAST_Compile` to generate a `PyCodeObject`, which is then fed into `run_eval_code_obj`. But does `_PyAST_Compile` compile *to* AST or *from* AST? (Note from future: the answer is *from*, read on for my reasoning.)

The function `pyrun_file` seems to have the answer. It returns yet another internal struct, something called a `PyObject` (not to be confused with a `PyCodeObject`), but what's promising is that its input is (finally) *not* some internal struct, but a straight-up file (hopefully a Python source file). I won't worry abt what `pyrun_file` returns for now: what I'm interested in is what's going on inside. First, `pyrun_file` calls some function called `_PyParser_ASTFromFile`, passing that file, and puts the result into a `mod_ty`. This confirms (?) that `mod_ty` indeed encodes Python AST. Then `pyrun_file` calls `run_mod` on that `mod_ty`. This gives us a way to execute a `mod_ty`, sweet. Third, looking at the body of `run_mod`, I see the answer to the question abt whether `_PyAST_Compile` compiles *to* AST or *from* AST. The answer is that it compiles *from* AST: I know this because `run_mod` takes a `mod_ty` (which I now know is AST) and passes it to `_PyAST_Compile` to generate a `PyCodeObject`. So I'm pretty sure that I don't have to worry abt the `PyCodeObject` type (which is compiled AST), the `_PyAST_Compile` function (which generates a `PyCodeObject`), or the `run_eval_code_obj` function (which runs a `PyCodeObject`).

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
Third meeting. CS 81c confirmed, which is welcome news since I only have like two more weeks of CS 81b. 

Created `pybr-parser-playground` branch with a new file, `playground.c`, where I'll play around with the parser internals (particularly the functions I found on Feb 18) to understand how to use them.

## Feb 25
Moved this journal to a GitHub Page within the pybr repo. Didn't like that because this journal is for me, not for other people working on pybr. Moved this journal to my personal GitHub Page. Turns out GitHub Pages specifically need to be in HTML, and I don't feel like writing this journal in HTML or routinely converting it to HTML or figuring out some other workaround. Decided to just move this journal to its own repo, with the journal acting as the repo's README. This works.

## Feb 27
I found the header file where the `mod_ty` type is defined, as well as the one where the `_PyParser_ASTFromString` function (and `_PyParser_ASTFromFile`, but that's later) is defined. So I'm able to use these symbols in `playground.c`. But `run_mod` isn't in any header files (I checked). I'm able to use it in `playground.c` if I `#include pythonrun.c`, but my understanding is that including `.c` files is a bad idea. So if there are symbols from the parser internals that I want to use in `playground.h`, but these symbols aren't in any header files, I guess I just... add them to a new header file? Or I can also just copy-and-paste the symbols I want into `playground.c`. Both feel hacky.

I'll worry about this later; having `_PyParser_ASTFromString` means that I *should* be able finally compile pybr source to AST, after I change and recompile the parser. And, yknow, figure out what exactly are the five `_PyParser_ASTFromString` parameters:

1. `char* str`. This one is easy: it's just the Python source to compile.
2. `PyObject* filename`. This one should be easy too, right? I'll just pass some dummy filename to something that gives me a `PyObject*` or whatever. Right? No. Almost every usage of a `PyObject` encoding a filename (I'm still not sure what's a `PyObject`, but it seems to be able to encode many different things) that I found gets the `PyObject` from somewhere else. I did find one instance of *creating* a `PyObject` from a filename string, but it used some absolutely *wacky* C macro nonsense. Specifically, the ritual involved the `_Py_DECLARE_STR` and `_Py_STR` macros from `pycore_global_strings.h`. So I tried to replicate the macro nonsense in `playground.c`, but couldn't get it to not give me a syntax error. Then I found a different potential approach: there's a function called `PyUnicode_FromWideChar` in `unicodeobject.c` that takes a `wchar_t*` and spits out a `PyObject*`. I presume that `wchar_t*` is a Unicode string, given the type's name and the name of the function I found it in. So it shouldn't be hard to convert a `char*` filename to a `wchar_t*` filename, then to use `PyUnicode_FromWideChar` to convert *that* to a `PyObject*` filename. Or, better yet, there's probably a function somewhere that takes me directly from `char*` to `PyObject*`. If I find it, I can skip the useless Unicode conversion. In any case, I'll get back to this `PyObject* filename` parameter later.
3. `int mode`. I looked around and found the compilation modes in `compile.h`. One of them is called `Py_file_input`; I'm technically compiling from a string (for now), not a file, but this compilation mode  makes the most sense (and everything so far suggests that compiling from a string delegates to compiling from a dummy file anyway). So I'll use `Py_file_input` as the compilation mode. And the only usage of `_PyParser_ASTFromString` that doesn't get the `mode` from somewhere else passes `Py_file_input` as the `mode`, so I'm pretty sure that's right.
4. `PyCompilerFlags* flags`. They're covered in Chapter 8.4 of the book, so I'll look at that later.
5. `PyArena* arena`. Same as above, but in Chapter 10.8.

## Feb 28
There's a lot to unpack in the parameters from the list above, and I'd rather have fewer things to unpack. Found two functions (`PyRun_SimpleStringFlags` in `Python/pythonrun.c`, and `pymain_run_command` in `Modules/main.c`) that maybe could've been useful for compiling a string of Python source with just the compilation flags, instead of having to figure out all this stuff about `PyObject` and `PyArena` and whatnot. The functions were not, in fact, useful. So let's get to unpacking.

`pyarena.c` and `pyarena.h`
