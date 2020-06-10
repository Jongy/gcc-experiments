GCC Experiments
===============

In this fork of GCC I'll work on some "expriments" and additions to GCC that I have thought of.

This is based on GCC 10.1.0.

Force inlining of a call site
-----------------------------

GCC accepts many types of hints regarding inlining of functions: the ``inline`` keyword, the
``always_inline``, ``flatten``, ``noinline`` attributes, and possibly more I forgot to list here.

Out of these methods, ``always_inline`` is the most obeyed one. Besides it, other inlinings are merely hints and
are affected heuristics, and anyway you should not be needed to force them.

Sometimes, however, finer control over which calls are inlined and which are not can be useful. It is easy to force
it upon the compiler using the preprocessor: write the function's code in a macro, then call the macro once in the
out-of-lined version of the function, and call it in all sites where you want it inlined as well.

It's just that... writing code inside macros sucks. It's by no means a good pattern. That's way I don't see it often
used.

What I did see lately, and triggered me to do this experiment, is
`this snippet <https://github.com/python/cpython/blob/3.9/Objects/object.c#L1211>`_ from CPython.
The developer decided it's worth to inline ``_PyObject_GetDictPtr`` in that site (obviously, it is best if such
desicions are made following some real performance tests and not by hunches). Since there wasn't any way (at least
none that I know of) to force GCC into inlining the function call (``_PyObject_GetDictPtr`` is used from many
sites, it's not a good candidate for inlining, size-wise) - the solution was to inline it "manually", that is,
copy the code of it and protect it with a comment ``/* Inline _PyObject_GetDictPtr */``.

Needless to say, this creates very fragile code. This is also not the first time I see this symptom in code, and
I think GCC can do better to help us.

So... We want some way to inline specific call sites of a function... Sounds like some extra info you *could* attach
with attributes. Too bad GCC doesn't allow attributes on function calls (only on function declarations, in that aspect).

What I did, then:

1. I added "function call attributes" to the C syntax. Basically they work like this: instead of ``func(5)`` you can now
   write ``func __attribute__((...)) (5)`` to place some attributes on this very call site.
2. Function calls respect (only) the attribute ``inline``, which is stored on the ``CALL_EXPR`` via ``CALL_EXPR_INLINE``.
3. During gimplification, the ``CALL_EXPR_INLINE`` sets the ``GF_CALL_INLINE`` flag on the new GIMPLE call statement.
4. In early inlining, call sites marked with ``GF_CALL_INLINE`` are treated as if the function was marked with ``always_inline``.
5. During LTO objects writing, ``GF_CALL_INLINE`` is translated to cgraph's ``force_inline_call``.
6. In LTO, ``force_inline_call`` is read from LTO objects, and marked functions are force-inlined in the first stage of IPA inlining.

That's it! Using this (experimental) GCC, I can selectively force the inlining of any function call.

You can test a small demo this way:

.. code-block:: c

    #include <stdio.h>

    void too_big(void) {
        // too big to be inlined automatically.
        printf("!!\n");
        printf("!!\n");
        printf("!!\n");
        printf("!!\n");
    }

    int main(void) {
        // these calls won't be inlined
        too_big();
        too_big();
        too_big();
        // this call will be
        too_big __attribute__((inline)) ();
        // and again, these won't
        too_big();
        too_big();
        too_big();

        return 0;
    }

Compile with ``-O2``.

Also, if you'd move ``too_big`` to another file and try again, it will work as well (this time via LTO, so enable ``-flto``).

TODOs:

1. Verify what the hell I did, this is merely the result of a few hours of work, I probably missed many spots.
2. To get closer behavior as with ``always_inline``, perhaps IPA inlining of ``force_inline_call`` should be done together
   with ``always_inline``, instead of as an extra step.
3. I'd want calls marked with "inline" to emit a compilation error if they fail to be inlined, for any reason.
4. This only works with optimizations enabled. Since this entire experiment is an optimization by itself, it's fair to
   assume this should only by used with optimizations enabled... But still, I'd want this to work even with ``-O0``, just
   like ``always_inline`` does.

Original GCC README
===================

This directory contains the GNU Compiler Collection (GCC).

The GNU Compiler Collection is free software.  See the files whose
names start with COPYING for copying permission.  The manuals, and
some of the runtime libraries, are under different terms; see the
individual source files for details.

The directory INSTALL contains copies of the installation information
as HTML and plain text.  The source of this information is
gcc/doc/install.texi.  The installation information includes details
of what is included in the GCC sources and what files GCC installs.

See the file gcc/doc/gcc.texi (together with other files that it
includes) for usage and porting information.  An online readable
version of the manual is in the files gcc/doc/gcc.info*.

See http://gcc.gnu.org/bugs/ for how to report bugs usefully.

Copyright years on GCC source files may be listed using range
notation, e.g., 1987-2012, indicating that every year in the range,
inclusive, is a copyrightable year that could otherwise be listed
individually.
