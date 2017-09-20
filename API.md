API Documentation
=================

* [Introduction](#introduction)
* [Overview](#overview)
  * [Create an Event Context](#create-an-event-context)
  * [Register an Event Watcher](#register-an-event-watcher)
  * [Start Event Loop](#start-event-loop)
  * [Summary](#summary)
* [Using -luev](#using--luev)
* [Joystick Example](#joystick-example)


Introduction
------------

The C API to [libuEv][], listed in `uev/uev.h`, handles three different
types of events: I/O (files, sockets, message queues, etc.), timers, and
signals.  The [Summary](#summary) details a slight caveat on signals.

Timers can be either relative, timeout in milliseconds, or absolute with
a time given in `time_t`, see `mktime()` et al.  Absolute timers are
called cron timers and their callbacks get an `UEV_HUP` error event if
the wall clock changes, either via NTP or user input.

**NOTE:** On some systems, embedded in particular, `time_t` is a 32-bit
  integer that wraps around in the year 2038.  A workaround for GLIBC
  systems (`-D_TIME_BITS=64`) protects against this problem, bu users of
  other C libraries have no known fix.  It is strongly recommended to
  use relative timers as often as possible.


Overview
--------

```C
/*
 * Callback example, arg comes from the watcher's *_init() function,
 * w->fd holds the file descriptor or socket, and events is set by
 * libuEv to indicate status: UEV_READ and/or UEV_WRITE with any of
 * the optional UEV_HUP, UEV_RDHUP, or UEV_PRI for urgent read data.
 */
void callback       (uev_t *w, void *arg, int events);

/* Event loop:      Notice the use of flags! */
int uev_init        (uev_ctx_t *ctx);
int uev_exit        (uev_ctx_t *ctx);
int uev_run         (uev_ctx_t *ctx, int flags);         /* UEV_NONE, UEV_ONCE, and/or UEV_NONBLOCK */

/* I/O watcher:     fd      *MUST* be non-blocking!
 *                  events  combination of the main flags:  UEV_READ, UEV_WRITE,
 *                                                          UEV_EDGE, UEV_ONESHOT
 */
int uev_io_init     (uev_ctx_t *ctx, uev_t *w, uev_cb_t *cb, void *arg, int fd, int events);
int uev_io_set      (uev_t *w, int fd, int events);
int uev_io_start    (uev_t *w);
int uev_io_stop     (uev_t *w);

/* Timer watcher:   schedule a relative timer, timeout and period in milliseconds */
int uev_timer_init  (uev_ctx_t *ctx, uev_t *w, uev_cb_t *cb, void *arg, int timeout, int period);
int uev_timer_set   (uev_t *w, int timeout, int period); /* Change timeout or period */
int uev_timer_start (uev_t *w);                          /* Restart a stopped timer */
int uev_timer_stop  (uev_t *w);                          /* Stop a timer */

/* Cron watcher:    schedule an absolute timer, when and period in time_t seconds */
int uev_cron_init   (uev_ctx_t *ctx, uev_t *w, uev_cb_t *cb, void *arg, time_t when, time_t period);
int uev_cron_set    (uev_t *w, int when, time_t period); /* Change when or period */
int uev_cron_start  (uev_t *w);                          /* Restart a stopped cron */
int uev_cron_stop   (uev_t *w);                          /* Stop a cron */

/* Signal watcher:  signo is the signal to wait for, e.g., SIGTERM */
int uev_signal_init (uev_ctx_t *ctx, uev_t *w, uev_cb_t *cb, void *arg, int signo);
int uev_signal_set  (uev_t *w, int signo);               /* Change signal to wait for */
int uev_signal_start(uev_t *w);                          /* Restart a stopped signal watcher */
int uev_signal_stop (uev_t *w);                          /* Stop signal watcher */
```


### Create an Event Context

To monitor events the developer first creates an *event context*, this
is achieved by calling `uev_init()` with a pointer to a (thread) local
`uev_ctx_t` variable.

```C
uev_ctx_t ctx;

uev_init(&ctx);
```


### Register an Event Watcher

For each event to monitor, be it a signal, timer or file descriptor, a
*watcher* must be registered with the event context.  The watcher, an
`uev_t`, is registered by calling the event type's `_init()` function
with the `uev_ctx_t` context, the callback, and an optional argument.

Here is a signal example:

```C
void cleanup_exit(uev_t *w, void *arg, int events)
{
    /* Graceful exit, with optional cleanup ... */
    uev_exit(w->ctx);
}

int main(void)
{
    uev_t sigterm_watcher;

    .
    .
    uev_signal_init(&ctx, &sigterm_watcher, cleanup_exit, NULL, SIGTERM);
    .
    .
}
```


### Start Event Loop

When all watchers are registered, call the *event loop* with `uev_run()`
and the argument to the event context.  The `flags` parameter can be
used to integrate [libuEv][] into another event loop.

In this example we set `flags` to none:

```C
uev_run(&ctx, UEV_NONE);
```

With `flags` set to `UEV_ONCE` the event loop returns as soon as it has
served the first event.  If `flags` is set to `UEV_ONCE | UEV_NONBLOCK`
the event loop returns immediately if no event is available.

**Note:** libuEv handles many types of errors, stream close, or peer
shutdowns internally, but also lets the callback run.  This is useful
for stateful connections to be able to detect EOF.

### Summary

1. Set up an event context with `uev_init()`
2. Register event callbacks with the event context using
   `uev_io_init()`, `uev_signal_init()` or `uev_timer_init()`
3. Start the event loop with `uev_run()`
4. Exit the event loop with `uev_exit()`, possibly from a callback

**Note 1:** Make sure to use non-blocking stream I/O!  Most hard to find
  bugs in event driven applications are due to sockets and files being
  opened in blocking mode.  Be careful out there!

**Note 2:** When closing a descriptor or socket, make sure to first stop
  your watcher, if possible.  This will help prevent any nasty side
  effects on your program.

**Note 3:** a certain amount of care is needed when dealing with APIs
  that employ signalfd.  If your application use `system()` you replace
  that with `fork()`, and then in the child, unblock all signals blocked
  by your parent process, before you run `exec()`.  This because Linux
  does not unblock signals for your children, and neither does most
  (all?)  C-libraries.  See the [finit][6] project's implementation of
  `run()` for an example of this.  For more details on this issue, see
  [this article][4] at [lwn.net](http://lwn.net).


Using -luev
-----------

LibuEv is by default installed as a library with a few header files, you
should only ever need to include one:

```C
#include <uev/uev.h>
```

The output from the `pkg-config` tool holds no surprises:

```sh
$ pkg-config --libs --static --cflags libuev
-I/usr/local/include -L/usr/local/lib -luev
```

The prefix path `/usr/local/` shown here is only the default.  Use the
`configure` script to select a different prefix when installing libuEv.

For GNU autotools based projects, use the following in `configure.ac`:

```sh
# Check for required libraries
PKG_CHECK_MODULES([uev], [libuev >= 1.4.0])
```

and in your `Makefile.am`:

```sh
proggy_CFLAGS = $(uev_CFLAGS)
proggy_LDADD  = $(uev_LIBS)
```


Joystick Example
----------------

Here follows a very brief example to illustrate how one can use libuEv
to act upon joystick input.

```C
#include <err.h>
#include <errno.h>
#include <stdio.h>
#include <stdint.h>
#include <fcntl.h>
#include <unistd.h>
#include <uev/uev.h>

struct js_event {
    uint32_t time;      /* event timestamp in milliseconds */
    int16_t  value;     /* value */
    uint8_t  type;      /* event type */
    uint8_t  number;    /* axis/button number */
} e;

static void joystick_cb(uev_t *w, void *arg, int events)
{
    read(w->fd, &e, sizeof(e));

    switch (e.type) {
    case 1:
        printf("Button %d %s\n", e.number, e.value ? "pressed" : "released");
        break;

    case 2:
        printf("Joystick axis %d moved, value %d!\n", e.number, e.value);
        break;
    }
}

int main(void)
{
    int fd;
    uev_t watcher;
    uev_ctx_t ctx;

    fd = open("/dev/input/js1", O_RDONLY, O_NONBLOCK);
    if (fd < 0)
        errx(errno, "Cannot find a joystick attached.");

    uev_init(&ctx);
    uev_io_init(&ctx, &watcher, joystick_cb, NULL, fd, UEV_READ);

    puts("Starting, press Ctrl-C to exit.");

    return uev_run(&ctx, 0);
}
```

To build the example, follow installation instructions below, then save
the code as `joystick.c` and call GCC

```sh
$ gcc `pkg-config --libs --static --cflags libuev` -o joystick joystick.c
```

Alternatively, call the `Makefile` with <kbd>make joystick</kbd> from
the unpacked [libuEv][] distribution.

More complete and relevant example uses of [libuEv][] is the FTP/TFTP
server [uftpd][5], and the Linux `/sbin/init` replacement [finit][6].
Both successfully employ [libuEv][].

Also see the `bench.c` program (<kbd>make bench</kbd> from within the
library) for [reference benchmarks][7] against [libevent][1] and
[libev][2].

[1]:      http://libevent.org
[2]:      http://software.schmorp.de/pkg/libev.html
[4]:      http://lwn.net/Articles/415684/
[5]:      https://github.com/troglobit/uftpd
[6]:      https://github.com/troglobit/finit
[7]:      http://libev.schmorp.de/bench.html
[LibuEv]: https://github.com/troglobit/libuev