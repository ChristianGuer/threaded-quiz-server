# Threaded Quiz Server

A multithreaded TCP quiz server in C. Clients connect over the network, wait in a
lobby until the group is full, then answer the same questions in lockstep while the
server tracks each player's score concurrently.

Written for Computer Systems II at James Madison University.

## Design

**Thread per client.** The accept loop spawns a thread per connection
([`qserver.c:299`](qserver.c#L299)), so one slow client never stalls the others.

**The first client becomes the admin.** Rather than hardcoding session size, the
server sends `QS|ADMIN` to the first connection and waits for it to reply with the
group size, then initializes the capacity semaphore with that value
([`qserver.c:125`](qserver.c#L125)). An `initialized` state variable guards the
window while that handshake is in flight, so clients racing in during setup are
handled rather than dropped.

**Capacity via a counting semaphore.** Later clients take a spot with `sem_trywait`,
which fails immediately when the group is full — the server answers `QS|FULL` and
the accept loop keeps running instead of blocking.

**Synchronized start via mutex and condition variable.** Players who arrive before
the group fills are sent `WAIT` and block on `start_cond`. When the last spot is
taken they are released together, so everyone sees each question at the same moment.
This is the part that makes it more than a thread-per-client echo server.

**CRLF line protocol.** Control messages are `\r\n`-terminated, pipe-delimited verbs
— `QS|ADMIN`, `QS|JOIN`, `QS|FULL` — defined in [`qserver.h`](qserver.h).

## Layout

| File | Role |
| --- | --- |
| `qserver.c` | Accept loop, admin handshake, session synchronization, scoring (427 lines) |
| `qclient.c` | Client: connects, joins a session, answers prompts (254 lines) |
| `qfuncs.c` / `qfuncs.h` | Question loading and formatting helpers |
| `qserver.h` | Protocol verbs and buffer sizing |
| `questions.txt` | Quiz content — numbered questions with lettered choices |
| `threaded_echo_server.c` | Earlier exercise: the thread-per-client pattern on its own |
| `connectsock.c` / `passivesock.c` | Socket setup helpers, based on W. Richard Stevens' networking code |

## Building and running

Developed on Linux with `gcc` and GNU `make`.

```bash
make                              # build qserver and qclient
./qserver questions.txt 5000      # serve questions.txt on port 5000

./qclient localhost 5000          # first client: becomes admin, sends group size
./qclient localhost 5000          # each additional player
```

## Attribution

`connectsock.c` and `passivesock.c` are adapted from the socket helper routines in
W. Richard Stevens' networking texts, as provided by the course. The server, client,
and question handling are my own.
