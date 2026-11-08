---
title: "Forking a CLI is easy. Making it behave like the CLI is the hard part"
permalink: /fork-fidelity/
description: "A pre-forked daemon makes a Python CLI start in milliseconds. Getting exactly what the real tool would have done took most of the work: environment, terminal, exit codes, state frozen at import, staleness, security."
---

# Forking a CLI is easy. Making it behave like the CLI is the hard part

In [the previous article](https://anthonygosme.github.io/) I showed where a Python CLI spends its time before doing any work, and one way around it for people who only use the tool: import it once in a daemon, and fork a copy for each call. That is what my tool, [speedrunner](https://github.com/anthonyGosme/speedrunner), does: `aws --version` goes from 386 ms to 40 ms.

I also wrote that speed was not the hard part. This is the hard part.

## The naive version

Here is a fork server that works, in the sense that it is fast:

```python
import os, runpy, socket, sys
import awscli.clidriver                      # the slow imports, done once

server = socket.socket(socket.AF_UNIX)
server.bind(path); server.listen()
while True:
    conn, _ = server.accept()
    if os.fork() == 0:
        fds, argv, cwd, env = receive(conn)  # the client's stdin/stdout/stderr, passed over the socket
        for i, fd in enumerate(fds):
            os.dup2(fd, i)
        os.chdir(cwd)
        os.environ = env
        sys.argv = argv
        runpy.run_path(script, run_name="__main__")
        os._exit(0)
    conn.close()
```

Run `aws --version` through it and you get the right line, ten times faster. Almost every line of it is wrong anyway. The rest of this article lists how, found one bug at a time, mostly by comparing every output with a normal run of the same command.

## Rule zero: never run a command twice

The first instinct with a daemon is to fall back to the real tool when something goes wrong. That is right, until the command has started. If the daemon dies halfway through `aws s3 rm`, running `aws s3 rm` again is not a fallback, it is a second deletion.

So the protocol has a point of no return. Before it, any problem (no daemon, a failed handshake, a stale cache) sends the call to the real tool, and the user sees nothing. The child says hello, sends its PID, and waits for a go. After the go, the client never runs the command again, whatever dies: if it loses track of the child, it says so and exits with a code of its own (125), rather than guess.

Every other decision in this article is made on the safe side of that line: if the child cannot reproduce the native behaviour, it must find out before the go, and hand the call back.

## The environment

`os.environ = env` is the first bug in the prototype, and a quiet one. Python code that reads `os.environ` afterwards sees the client's values. But libraries keep references to the old mapping (botocore does), and C code calls `getenv()`, which reads the process environment, still the daemon's. So does every subprocess the tool starts. The child has to change `os.environ` in place: `clear()` then `update()`, which call `unsetenv` and `putenv` underneath. quicken, one of the earlier fork servers for Python, reassigns it.

That is not enough either, because some modules read the environment while they are imported, in the daemon, and keep the value. botocore does this for a retry setting and for a path built from `HOME`; gettext fixes the language. The daemon cannot know in advance which variables matter, so it records them: during the imports, every read of `os.environ` is noted. At each call, if the client's value for one of those keys differs from the daemon's, the call goes native. Reads made with `getenv()` from C, at import, still escape this.

The opposite direction matters too: some imports set variables. The daemon records the difference the imports made, and the child applies it again on top of the client's environment.

And a small one: a meson module tries `os.environ[b"X"]` inside a `try`, which raises natively. The recorder had noted the key anyway, then sorted a list mixing `str` and `bytes`, and every child failed before the go. Nothing broke for the user, since that is the safe side of the line, but meson, pio and copier quietly ran at native speed until it was found.

## The terminal

This is where the bugs were the most visible.

**Streams.** Passing the client's file descriptors over the Unix socket (`SCM_RIGHTS`) and `dup2`-ing them onto 0, 1 and 2 gets the bytes right. But Python decided how to buffer `sys.stdout` when the daemon started, with no terminal. Recreating the stream objects would break the references that modules took at import (a `logging` handler holds the old `sys.stderr`). So the child reconfigures `sys.stdout` in place, with `reconfigure(line_buffering=...)`, according to what the client's descriptors are. (CPython always line-buffers stderr.)

**`isatty()` at import.** httpie, ansible (for its colours) and rubocop (through a gem) ask once, at import, whether their streams are terminals, and remember the answer. The daemon has no terminal, so the answer was no, for every call. No automated comparison saw it, because the tests captured output and had no terminal on either side. The recorded demo did: httpie failed, ansible and rubocop lost their colours. Now the daemon imports with a pseudo-terminal on each descriptor that was a terminal for the client that started it, records every `isatty` question, and a client in a different situation runs natively.

**`/dev/tty`.** A forked child has no controlling terminal. `getpass` opens `/dev/tty` to read a password without echo; when that fails, it falls back to stdin. Served, with stdin redirected, it read the password from the pipe and showed the one you typed in clear. The client now sends its own terminal descriptor too, and the child answers any open of `/dev/tty` with a copy of it. A subprocess the tool starts that opens `/dev/tty` itself (ssh, sudo, git asking for a password) still fails: that is a known limit, not a fix.

## Exiting like CPython

`os._exit(0)` is the prototype's next lie. A Python program's exit code comes from `SystemExit`: `sys.exit()` and `sys.exit(None)` give 0, an integer gives itself, and anything else is printed on stderr and gives 1. An uncaught exception gives 1 with its traceback. `atexit` handlers run, buffers are flushed. JumpTheGun, another prior tool, returned 1 for `sys.exit(None)`; quicken fixed that case but not the string one.

Then there is the cost of leaving. Tearing an interpreter down takes time: about 43 ms for aws v1 on my test server. The child does not need the client to wait for that. It flushes what teardown would have flushed (the streams, and buffered files still open, found through the garbage collector), sends the exit code, points its standard descriptors to `/dev/null` so that a pipeline sees its end, and finishes tearing down alone. That saved 37 ms per call.

It is not free. If flushing fails, the child sends no code, and the client waits for the real exit, 120, as Python gives natively. If an object with a `__del__` method or a suspended generator is still alive, teardown may still run the tool's code, so the child says so and the client waits. What remains invisible is small and listed: a message printed by a weak reference callback during teardown, for instance.

## State frozen at import

The daemon runs the tool's imports once, for every future call. Anything those imports capture is captured for all of them.

- **`sys.argv`.** httpie defines `def main(args=sys.argv)`: the default is the list object that existed at import. Assigning a new list to `sys.argv` in the child leaves httpie with the daemon's arguments. The child changes the list in place, `sys.argv[:] = ...`.
- **A program that runs when imported.** Some modules do the work as they load: Ruby's `rails/cli` is one. The daemon notes reads of the command line during imports; a module that reads more than the program name is not preloaded.
- **Import hooks installed later.** pytest rewrites assertions through an import hook it installs at run time. Plugins preloaded before that hook existed were not rewritten, and pytest said so. Only modules loaded by standard loaders are preloaded now.
- **Threads.** `fork()` copies one thread. A thread started at import, in the daemon, does not exist in the child, and the locks it held stay locked. The daemon refuses to fork unless exactly one thread runs, checked both in Python and in `/proc/self/task`, so a native thread counts too. A Ruby gem used by kamal and capistrano starts a cleanup thread when loaded: that file is now left out of the preload. On macOS, the same rule guards against CoreFoundation: on Apple Silicon, a child that touched it after the parent had used it was killed, 20 times out of 20. Using it leaves threads running in the parent, so the rule refuses to fork.
- **Time.** A clock read at import is the daemon's. rspec prints how long its files took to load, measured from a clock read at load; an adapter resets it to the start of the call.
- **The working directory.** prettier, az and bundler keep the directory they started in. They are served only from that directory, and run natively elsewhere.
- **Hashing.** Every child shares the daemon's hash seed, where native runs each draw their own. `random` and `uuid` stay random. But anything whose order depends on the process comes out in the same order every time: with four plugins installed, `llm plugins` lists them in 4 different orders over 15 native runs, and in one order over 15 served runs. Same content; but a script that compares two runs would see the native order change and the served one stay put.

## Knowing when the cache is stale

A daemon that outlives `pip install -U` serves the old version. The obvious check is a fingerprint: the interpreter and its version, the script, the installed packages' records, and, since a forked child inherits the daemon's identity rather than the client's, the user, group and supplementary groups.

Then a test failed: a module edited in place, same size, same inode, in a directory whose time had not changed, was served stale. Checking the modification time of every loaded file covers it, but aws loads 601 of them, and a `stat` of each cost 1.4 ms per call, doubling the handshake. Instead, the daemon watches every loaded file with inotify, and nobody ever reads the event queue: a pending event means stale, for every process that looks. Each call costs one `select()` on it. A stale fingerprint is found before the go, so the call runs natively, and the daemon restarts in the background.

## Security: whose daemon is this?

A daemon that runs your tools with your environment is a target: your environment holds your cloud credentials. Two rules.

**Never a network port.** Only a Unix socket, in a directory that must belong to you and have mode exactly 0700. The check is made on the opened directory itself (`fstat` on its descriptor), and the daemon then works relative to it, so the directory cannot be swapped between the check and its use. JumpTheGun's fallback in `/tmp` reuses the first directory matching its name pattern without checking its owner: another user could create it first and receive the client's environment.

**Check the other end.** With `SO_PEERCRED`, the daemon checks that the client is your user, and the client checks that the daemon is. If either check fails, the call runs natively.

## One daemon per context

Some things a child cannot take from its client at all, because they belong to the process from its start: resource limits, priority, CPU affinity, namespaces, capabilities, library paths, the interpreter a `#!/usr/bin/env python3` line found on the client's `PATH`. Rather than turn such clients away, these go into the daemon's name: a client with a different context gets a different daemon. A few cases have no answer and always run natively: `LD_PRELOAD`, a macOS sandbox, a descriptor above 2 inherited from the shell, a background job.

The same idea fixed a quieter bug. `./manage.py` was passed to the daemon as typed: two Django projects would have shared one daemon. The daemon now gets the absolute path, one per project, while the tool still sees `./manage.py` in its arguments.

## How these were found

Almost none of them by thinking hard. By comparison: every command run natively and served, and the exit code, stdout and stderr compared byte for byte. The first run over a panel of real tools had five differences: four were bugs (httpie's arguments, rails, pytest's hook, and one in PHP). A second panel of fifty more tools found more, including the `bytes` key. A stress test that killed and restarted daemons under parallel calls found daemons that exited without waiting for their children, leaving clients to report a lost command that had in fact run.

Each bug became a test. The Python matrix alone has 219 of them now, and all 96 commands of the published benchmark give the same output as a native run.

What the comparison cannot see is everything it does not capture: a terminal (the `isatty` bugs), a password prompt, a file written on the side. So the rule has a corollary: test in the conditions people use the tool in, not only in the ones that are easy to capture.

## A checklist, if you build one of these

- Decide where the point of no return is, and never cross it twice.
- Change `os.environ` in place. Record what imports read and set.
- Reconfigure the standard streams in place; answer `isatty` the way the client's terminal would; give the child the client's `/dev/tty`.
- Reproduce `SystemExit` exactly, run `atexit`, flush before reporting the code.
- Mutate `sys.argv` in place. Refuse to fork with more than one thread, native ones included.
- Assume everything read at import is frozen: environment, terminal, clock, working directory, hash seed.
- Invalidate on every file the daemon loaded, not only on package versions.
- Unix socket, owner and mode checked on the descriptor, peer credentials checked both ways.
- Put what a child cannot inherit into the daemon's identity.
- Compare every output with a native run, in a terminal too.

speedrunner does all of this for Python, Ruby, PHP and Node tools, and its [known limits](https://github.com/anthonyGosme/speedrunner/blob/main/docs/GUIDE.md#known-limits) list what it still cannot reproduce. The code is [on GitHub](https://github.com/anthonyGosme/speedrunner).
