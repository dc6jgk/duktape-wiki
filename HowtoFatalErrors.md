# How to handle fatal errors

## Error types

There are three types of errors handled by Duktape:

- Ordinary errors caused by e.g. `throw`, `duk_error()`, and `duk_throw()`.
- Fatal errors caused by e.g. an uncaught error, an explicit call to
  `duk_fatal()`, or an unrecoverable error inside Duktape.
- Fatal errors without any Duktape heap or thread context, caused by e.g.
  assertion failures inside Duktape.

Normal errors are propagated internally using a `longjmp` or C++ exceptions
(depending on configuration).  They are caught using an ECMAScript `try-catch`
or a [protected](http://duktape.org/api.html#taglist-protected) C API call.

Fatal errors caused by e.g. uncaught errors trigger a call into a fatal error
handler which is registered in `duk_create_heap()`.  If the handler argument
was given as `NULL` a built-in default fatal error handler is used instead.
The default fatal error handler either calls `abort()` or throws a C++
exception (`duk_fatal_exception`) depending on the `DUK_USE_CPP_EXCEPTIONS`
config option.
**It is strongly recommended to provide a custom fatal error handler**.

Fatal errors without context, currently limited to assertion failures, always
trigger a call into the built-in default fatal error handler because a heap
related handler cannot be looked up without context.  These only happen if
you explicitly enable assertions.

## Example fatal error handler

Registering a handler during heap creation:

```c
duk_context *ctx;
void *my_udata = (void *) 0xdeadbeef;  /* whatever's most useful, can be NULL */

ctx = duk_create_heap(NULL, NULL, NULL, my_udata, my_fatal);
/* ... */
```

The fatal error handler can for example:

```c
static void my_fatal(void *udata, const char *msg) {
    (void) udata;  /* ignored in this case, silence warning */

    /* Note that 'msg' may be NULL. */
    fprintf(stderr, "*** FATAL ERROR: %s\n", (msg ? msg : "no message"));
    fflush(stderr);
    abort();
}
```

In Duktape 1.x the fatal error handler function had an additional error code
argument.

## Built-in default fatal error handler

The built-in default fatal error handler is optimized for minimal assumptions
about the underlying platform.  The default behavior depends on the
`DUK_USE_CPP_EXCEPTIONS` config option:

- If `DUK_USE_CPP_EXCEPTIONS` is disabled (default): the default handler writes
  a debug log entry (but **does not write anything to stdout or stderr**!) and
  then calls `abort()` through the `DUK_ABORT()` config define (which can be
  replaced).  If the abort() call returns, the handler enters an infinite loop
  to make sure execution doesn't proceed after a fatal error (this should
  obviously never happen).

- If `DUK_USE_CPP_EXCEPTIONS` is enabled: the default handler writes a debug log
  entry (but does not write anything to stdout or stderr) and then throws a
  `duk_fatal_exception`.  The exception inherits from `std::runtime_error` and
  provides a `::what()` method to access the fatal error message.  Even though
  the exception is catchable, it is still unsafe to continue execution after
  catching it.

- Before Duktape 2.3 the behavior for `DUK_USE_CPP_EXCEPTIONS` was a bit
  different: uncaught errors would propagate out as `duk_internal_exception`,
  while other fatal errors (such as assertion failure) would use `abort()`.

Because the default behavior is not very useful in most environments, you should:

1. Provide a fatal error handler when creating a heap.  This is good practice
   because you then control how fatal errors (except for assertion failures)
   are dealt with.
2. Override the built-in default fatal error handler in `duk_config.h`.  This
   improves fatal error handling for Duktape heaps without an explicit fatal
   error handler, and for fatal errors triggered without context (such as
   assertion failures).  Overriding the default handler is especially important
   when providing Duktape as a system library.

Example of overriding the default fatal error handler in `duk_config.h`:

```c
/* ensure 'stdio.h' is included */

#define DUK_USE_FATAL_HANDLER(udata,msg) do { \
        const char *fatal_msg = (msg); /* avoid double evaluation */ \
        (void) udata; \
        fprintf(stderr, "*** FATAL ERROR: %s\n", fatal_msg ? fatal_msg : "no message"); \
        fflush(stderr); \
        abort(); \
    } while (0)
```

You can provide the handler also using genconfig ``--option-file`` like:

```
# my_fatal.yaml
DUK_USE_FATAL_HANDLER:
  verbatim: |
    #define DUK_USE_FATAL_HANDLER(udata,msg) do { \
            const char *fatal_msg = (msg); /* avoid double evaluation */ \
            (void) udata; \
            fprintf(stderr, "*** FATAL ERROR: %s\n", fatal_msg ? fatal_msg : "no message"); \
            fflush(stderr); \
            abort(); \
        } while (0)
```
