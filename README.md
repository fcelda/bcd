# Introduction

BCD is a simple library for invoking out-of-process tools in response to
program errors. By default, BCD is configured to use the Backtrace I/O
tracers and database clients to allow for process snapshots to be directly
submitted to the database.

If you are interested in enabling BCD without modifying source-code, please
refer to the "Preload" section of the document.

## Supported Platforms

BCD currently only supports Linux, but FreeBSD and IllumOS support is planned.
GCC, clang and ICC are supported. Though BCD must be built with a C compiler
that supports GNU99, the BCD interface is C++ compatible.

## Dependencies

For glibc versions before 2.17, librt is required.

## Build

To use BCD as a stand-alone library, use:

1. `./configure`
2. `make`
3. `make install`

It is possible to use BCD in amalgamated mode so that you only need to drop
in a header and source file in your source tree. In order to do this, you would
do the following:

1. `./configure`
2. `make amalgamated`

The source file is in `src/bcd-amalgamated.c` and the header file is
`include/bcd.h`. Just drop those in your source-tree.

## Usage

For a simple example, please look at `regressions/broad.c`. For more detailed
documentation, please refer to `include/bcd.h`.

In order to initialize the library, you must call `bcd_init`. This will
initialize the library for use. If you would like to emit non-fatal errors,
then you must also use `bcd_attach` to initialize a `bcd_t` object. Every
thread must use a unique `bcd_t` object.  These `bcd_t` objects may be
destroyed with `bcd_detach`.

You may use `bcd_emit` to request a snapshot of a non-fatal error. The error
will be grouped according to the error message passed to `bcd_emit`. For
fatal errors, you may call `bcd_fatal`. BCD will exit and deinitialize after
a call to `bcd_fatal`, so it is expected that your program will exit at this
point.

Credentials, permission, OOM killer interaction and more may be configured with
the `bcd_config` data structure, please see `bcd.h` for details.

## Troubleshooting

If you are on Linux and yama is enabled, you are able to change ptrace scope
for your program using `prctl`.

```
#include <sys/prctl.h>

prctl(PR_SET_PTRACER, PR_SET_PTRACER_ANY, 0, 0, 0);
```

System-level controls are otherwise available in `/proc/sys/kernel/yama`.

For more details, please see:
    https://www.kernel.org/doc/Documentation/security/Yama.txt

### Example

```c
#include <bcd.h>
#include <stdio.h>
#include <stdlib.h>

int
main(void)
{
	struct bcd_config config;
	bcd_error_t error;
	bcd_t bcd;

	/* Initialize BCD configuration. See bcd.h for options */
	if (bcd_config_init(&config, &error) == -1)
		goto fatal;

	/* Initialize the library. */
	if (bcd_init(&config, &error) == -1)
		goto fatal;

	/* Initialize a handle to BCD. This should be called by every thread interacting with BCD. */
	if (bcd_attach(&bcd, &error) == -1)
		goto fatal;

	if (bcd_kv(&bcd, "application", "my_application", &error) == -1)
		goto fatal;

	if (bcd_kv(&bcd, "datacenter", "my data center", &error) == -1)
		goto fatal;

	/*
	 * Generate a snapshot of the calling thread and upload it to
	 * the Backtrace database. Key-value attributes will be included.
	 */
	bcd_emit(&bcd, "This is a non-fatal error");

	/*
	 * We generate a fatal error and exit immediately.
	 */
	bcd_fatal("This is a fatal error. No recovery");
	return 0;
fatal:
	fprintf(stderr, "error: %s\n",
	    bcd_error_message(&error));
	exit(EXIT_FAILURE);
}
```

### Limitations

For ease of use, `bcd_emit` and `bcd_fatal` do not provide return values.
Error callbacks are used which are executed in the context of the BCD slave.
A developer may wish to register their own callbacks if they wish to terminate
their process due to error logging failures. BCD will never terminate the
calling process itself.

BCD relies on a UNIX socket (by default, `/tmp/bcd.*`) for thread communications
and relies on pipes for handling fatal errors and initial library configuration.
If either of these facilities are erroneously closed, a fatal error will be
generated by BCD, and BCD will exit. If this is a concern, then it is
recommended to install your own error handler to kill the process that is being
monitored.

BCD currently allocates memory in a separate process on errors, which may not
be suitable for systems that lack overcommit.

### Design

BCD will initialize a pipe and fork a process during `bcd_init`. This child will
initialize a UNIX socket for per-thread communications. The pipe is used early
in initialization to communicate configuration errors and is used for fatal
error handling. The BCD child process will fork and exec in response to trace
requests.

All communication between the host application and BCD occurs in a synchronous
fashion, for reasons of correctness. There is a global ordering to all BCD
operations.

### Handling Crashes

BCD tries to minimize modifying the program run-time environment. Complex
run-times may rely on signals for functionality. This means BCD does not, by
default, set any signal handlers. In order to handle crashes, ensure you
install a signal handler. You are able to use `bcd_emit` for recoverable
crashes and `bcd_fatal` for non-recoverable crashes. These functions are
signal-safe and thread-safe.

Below is a simple example that utilizes `signal`. For production use,
please use `sigaction` with `SA_SIGINFO` set, this allows for additional
data to be extracted at time of fault.

```c
#include <bcd.h>
#include <signal.h>
#include <stdlib.h>
#include <unistd.h>

static void
signal_handler(int s)
{

	bcd_fatal("This is a fatal crash");
	raise(s);
	return;
}

int
main(void)
{
	struct bcd_config config;
	bcd_error_t error;
	bcd_t bcd;

	/* Initialize BCD configuration. See bcd.h for options */
	if (bcd_config_init(&config, &error) == -1)
		exit(EXIT_FAILURE);

	/* Initialize the library. */
	if (bcd_init(&config, &error) == -1)
		exit(EXIT_FAILURE);

	/* Initialize a handle to BCD. */
	if (bcd_attach(&bcd, &error) == -1)
		exit(EXIT_FAILURE);

	if (signal(SIGSEGV, signal_handler) == SIG_ERR)
		abort();

	if (signal(SIGABRT, signal_handler) == SIG_ERR)
		abort();

	return 0;
}
```

# Preloading

It is possible to `LD_PRELOAD` BCD into an application. The preloading will
initialize BCD in the target application through a library constructor,
initializing signal handling and routing crashes to BCD. This is useful
in container environments or situations where you want to enable Backtrace
on an application without modifying the source-code.

Example usage:
```
$ BCD_PRELOAD=1 LD_PRELOAD=/opt/backtrace/lib/libbcd_preload.so ./program
[BCD] Initializing BCD...
program.264261.1605471835.btt
Aborted (core dumped)
```

You must include the path to `libbcd_preload.so` in `LD_PRELOAD` environment
variable and set the `BCD_PRELOAD` environment variable to `1` to enale
preloading.

The following environment variables can be used to tune BCD configuration:

```
       BCD_INVOKE_PATH: Path to the program to invoke on a fatal signal.
         BCD_INVOKE_KP: Key-value pair option.
  BCD_INVOKE_SEPARATOR: The separator to use between key-value pairs.
         BCD_INVOKE_KS: The seperator to use for a key-value pair.
         BCD_INVOKE_TP: The seperator to use or threads.
BCD_INVOKE_OUTPUT_FILE: Specifies the output file for the BCD_INVOKE_PATH program.
```

