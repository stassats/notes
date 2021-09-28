It's not always easy to find concrete things to change in SBCL. I
often stumble upon them by acccident.

So, here's how fixing one bug turned out into a series of commits.


Recently, Eric Marsden reported a bug that
```
(loop for i below 3 sum
        (ldb (byte 6 6)
             (ash i (mask-field (byte 5 8) i)))))
```
triggers an assertion in an `ash` VOP (the final product of compilation,
one step from being turned into machine code) which said "shifting by
0".
So I thought, "I know this category of bugs, something is not
optimized away in the earlier stages, should be easy to fix, or
failing that, just remove the assertion". 

`(mask-field (byte 5 8) i)` is always zero, because `i` is between 0 and 2.
But, before the compiler derives that fact, the `ash` function is
transformed into a VOP directly in the middle of the intermediate
stages of compilation, which is a bad idea as it will become opaque
and unoptimizable. So the fix is easy, remove that transform, add a
new transform for `(ash-left x 0) => x` and make the existing VOPs
do the job of that bad transform.
Here's the [commit](https://github.com/sbcl/sbcl/commit/02a200e80c51c782d1113bb7bfc4923652b6b117).

But then I noticed that 
while `(logand #xF00 (the (integer 0 2) x))` derives to `(integer 0 0)`
the call to the AND instruction is still present:

```
CMP R0, #4 ;; 4 because that's a tagged fixnum 2
BHI L0
AND R0, R0, #7680
```

SBCL folds functions with constant arguments into constants, but here
the arguments are not constant, just the result.

Changing that is
[simple](https://github.com/sbcl/sbcl/commit/02a200e80c51c782d1113bb7bfc4923652b6b117),
replacing a function that was just derived to a constant with that
constant.

But then ansi-tests fails saying that `(imagpart (the short-float x))`
returns `0.0` on `-1.0`, while it should be `-0.0`.
Ok, changing `imagpart` to derive `(single-float -0.0 0.0)`, but then
Christophe Rhodes points out that `(single-float 0.0 0.0)` actually
encompasses `-0.0`. 
Have to make sure `type-singleton-p` returns `NIL` on `(single-float 0.0 0.0)`:
[commit](https://github.com/sbcl/sbcl/commit/27b1003e738809a3628bea6eef074a81467fa3c2).

Next, 
```
(logand #xFF (1+ (the (integer 1 4) x)))
compiles to

ADD R0, R0, #2
AND R0, R0, #510
```

yet the result is within the first 8 bits, no need to apply that AND mask.
This is modular arithmethic, which avoids allocating bignums if the
high bits are unused, i.e. cut off by LOGAND.
A special LOGAND transform goes through its arguments and makes sure
they all are using functions that cut off high bits. But these
functions didn't have proper type derivers, so after transforming `(1+
(the (integer 1 4) x))` was just assumed to be
`FIXNUM`. [Rectified that](https://github.com/sbcl/sbcl/commit/9852ce2d678c4df2ffc3142f3ad7e8d3c34bf878)

Then I'm poking more around modular arithmethic, and see that 
`(logand #xFF (ash (the fixnum x) (the fixnum shift)))`
has a full call to ASH.
Even though either `(the (and fixnum (integer * 0)) shift)` or `(and
fixnum (integer 0 *))` are inlined.
Which got
[corrected](https://github.com/sbcl/sbcl/commit/a9dc095ff7cf357de3504233c89a477468b959a5)
by adding a new modular arithmethic function which works on both
shifting directions.

In the end, a single bug report turned into multiple commits making
several improvements.
