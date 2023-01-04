# OpenGL on Linux with Xlib

Or, How to slowly descend into the madness that is Xlib.

> This is a re-upload of a blog post I wrote back in 2020-02-09 with
> an anonymous account that I recently removed. I re-upload here in
> the hopes that someone will find it useful.
>
> 2023-01-04

## Purpose

Any self-respecting game engine needs the ability to render things
while doing other work at the same time. In particular, since fast
reaction to input is vital in many games it is absolutely necessary to
be able to read input as soon as possible, and that might very well be
at the same time as we are rendering our frame.

## Why Xlib?

As you try to use Xlib and get weird obscure errors that you cannot
really trace correctly to any point of origin, you will eventually
stumble upon this wonderfully titled page [I hate Xlib and so should
you](https://www.remlab.net/op/xlib.shtml). I do not know how but this
author manages to almost perfectly embody the experience of what it is
like to do pretty much anything in Xlib.

Okay, so Xlib is kind of yuck, but what should we use instead? Well,
we could try and use the suggested alternative, XCB. That is probably
better, right? After all, not even the official documentation for X
recommends that you use Xlib.

> For low-level X development, XCB, the X C Bindings provide a clean
> low-level protocol binding. Its older cousin Xlib (or libX11), is
> not recommended for new development, but is still very widely used.
>
> https://www.x.org/wiki/Documentation/

XCB sounds great! What is there not to like? XCB is closer to the
source, has a lot better error handling, and Xlib is in part
implemented on top of XCB. In fact, some of the errors I have gotten
using Xlib have had tracebacks into XCB. Let’s start using XCB!

> To use OpenGL in the X Windowing system, one must use the GLX API,
> and the GLX API is closely coupled with Xlib. As a result, an OpenGL
> application on the X Windows must use Xlib and thus can’t be done
> using only XCB.
>
> https://xcb.freedesktop.org/opengl/

![KEKWait](https://cdn.frankerfacez.com/emoticon/390750/4)

So there you have it. It is possible to kind of use XCB but you will
never be able to completely remove the dependency to Xlib, yet. And
since we are making a game engine here and not a proper GUI
applications with buttons and shit, I am not convinced that mixing XCB
and Xlib will help us that much. So Xlib it is!

## First attempt

Simply making X function calls from two threads without any
synchronization between them does not work due to Xlib not being
thread safe. “Ah, but there is a function XInitThreads you need to
call first” you might be thinking. Alas, it is not enough, since all
it does is adding locking mechanisms to all calls you make to X. There
is a more pressing issue we have to deal with first.

The separation we want to make is to have all graphics calls in one
thread while some other thread reads input events. How do we read
those events? If we simply call XNextEvent it will lock the display
and wait, prohibiting any graphics calls to be made in the other
thread! The option to query for events at some regular interval is a
non-option since the whole point of the event thread is to handle the
events as soon as possible.

After surfing the webs about how to separate the graphics calls to its
own thread, you might stumbled upon the alluring idea of having two
XDisplays, one for graphics calls in the render thread, the other for
events in the main thread (see [Xlib and openGl in
multithreading](https://community.khronos.org/t/xlib-and-opengl-in-multithreading/63159),
or for a complete example [Xlib + openGL programming and
multithreading](https://rosario3d.wordpress.com/2011/01/24/xlib-opengl-programming-and-multithreading/)). If
you try it, it might work. I did, and it worked for me. Happy
days. Until one day, many years later when I started writing tests, it
just stopped working. The tests were all failing but the application
could still run normally.

I managed to trace the failing tests down to the call of XSelectInput,
the one that you can use to instruct the render window to send all its
input events to the event display. It turns out I was not the only one
with problems (see [XSetWMProtocols and glXCreateContext calling order
in a Multithreaded
environment](https://stackoverflow.com/q/11272054)).

I can only assume as much as the window must be created from the same
display as you redirect the events to. The man pages does not really
add anything useful to the matter either.

> `int XSelectInput(Display *display, Window w, long event_mask);`
>
> display – Specifies the connection to the X server.
> event_mask – Specifies the event mask.
> w – Specifies the window whose events you are interested in.
>
> https://www.x.org/releases/X11R7.7/doc/man/man3/XSelectInput.3.xhtml

(I have a vague memory about reading somewhere that the window has to
be created from the same display as the call to XSelectInput but I
seem to have lost that reference. And even so, why did it seem to work
for such a long time?)

## Worse attempt

Slowly be convinced by all comments you read on the internet about not
trying to separate graphics from the main application, that having
everything jammed together in one giant loop is not a sign of poor
design and bad user experience as your users slowly get the feeling
that your program is slow and sluggish (they cannot really pin point
exactly where the problems are, it is more of an overall feeling), but
is instead a strength due to its robustness and simplicity.

## Better attempt

Read an answer from Stack Overflow user 34088 (see [how to quit the
blocking of xlib's XNextEvent](https://stackoverflow.com/a/8592969))
and get informed about there being something called the *connection
number* for XDisplays, and that there even is a macro that fetches it
for you, with the explanation:

> On a POSIX-conformant system, this is the file descriptor of the
> connection.
>
> https://tronche.com/gui/x/xlib/display/display-macros.html

A breakthrough! Now we can have the main thread simply wait for
anything incoming to the connection number of the display, parse it,
and go back to wait for more. However, we need to be careful. Every
time there are new events to read, this thread will wake up, but not
all events are meant for this thread. We have to guard ourselves from
that by making sure that there really are events to be read. The good
thing is that if the event is meant for someone else than us then they
will already have taken care of it at the time we check if there are
any events. An implementation that does all this can be seen below.

```c++
int message_loop(Display * display)
{
    struct pollfd fds[1] = {
        {ConnectionNumber(display), POLLIN, 0}
    };

    while (true)
    {
        const auto ret = poll(fds, 1, -1);
        if (ret == -1)
        {
            if (errno == EINTR)
                continue;

            return ret;
        }

        if (XPending(display) == 0)
            continue;

        while (XQLength(display) != 0)
        {
            XEvent event;
            if (XNextEvent(display, &event))
                return -1;

            if (XFilterEvent(&event, None) == True)
                continue;

            switch (event.type)
            {
            case ButtonPress:
                ...
            }
        }
    }
}
```

Line 4 is where we get the connection number, and line 9 is where we
go to sleep if there are no incoming events right now. Line 18 is our
guard that checks if there actually are any events to read. It will
synchronize with the X server to get an up to date number. Line 21
will also check, but it does so without syncing, and so is much faster
to have in a loop. Line 24 could block, but it will not since we know
there will be an event ready for us. Finally, line 27 does some fancy
filtering stuff that you may or may not need in your implementation.

Now, if you combine this approach of reading input events with
XInitThreads then you have a truly multithreaded way of working with
X11 and OpenGL. Good Luck!
