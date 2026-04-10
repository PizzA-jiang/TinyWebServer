# AGENTS.md

## Project at a glance
- `TinyWebServer` is a Linux/POSIX C++ web server built around `epoll`, `pthread`, non-blocking sockets, a timer list, a MySQL connection pool, and a singleton logger.
- The startup path is `main.cpp` -> `config.h` / `config.cpp` (`Config::parse_arg()`) -> `WebServer::init()` -> `log_write()` -> `sql_pool()` -> `thread_pool()` -> `trig_mode()` -> `eventListen()` -> `eventLoop()`.

## Read first
1. `README.md` for the supported runtime flags and the intended request flow.
2. `main.cpp` plus `config.h` / `config.cpp` for boot order, CLI flags, and default DB credentials.
3. `webserver.h` / `webserver.cpp` for server lifecycle, epoll setup, signal handling, and timer integration.
4. `http/http_conn.h` / `http/http_conn.cpp` for request parsing, routing, and response generation.
5. `threadpool/threadpool.h`, `CGImysql/sql_connection_pool.*`, `log/log.*`, `timer/lst_timer.*`, `lock/locker.h`.

## Runtime architecture
- `WebServer` owns the shared state: `users[]`, `users_timer[]`, `m_connPool`, `m_pool`, `m_epollfd`, and the listening socket.
- Signals are bridged into the event loop through `socketpair()`; `SIGALRM` drives `Utils::timer_handler()`, and `SIGTERM` stops the loop.
- Timers are stored in `sort_timer_lst`; `cb_func()` closes stale sockets and decrements `http_conn::m_user_count`.

## Request processing rules
- `http_conn::parse_request_line()` maps `GET` and `POST`; `POST` enables CGI-style login/register handling via the request path.
- Route handling in `http_conn::do_request()` is hardcoded to project pages under `root/`:
  - `/` -> `judge.html`
  - `/0` -> `register.html`
  - `/1` -> `log.html`
  - `/2` and `/3` -> login/register handlers
  - `/5`, `/6`, `/7` -> `picture.html`, `video.html`, `fans.html`
- Do not rename files in `root/` or change these URLs without updating the handler logic.

## Data and concurrency patterns
- Database access uses `connection_pool::GetInstance()` plus `connectionRAII` for scoped checkout/release.
- `threadpool<T>` switches behavior with `m_actor_model`: `1` = Reactor (`append`/`m_state`), otherwise Proactor (`append_p`).
- The global `users` map in `http/http_conn.cpp` caches usernames/passwords loaded from `SELECT username,passwd FROM user`.
- Logging is a singleton (`Log::get_instance()`); async mode is enabled when `Log::init(..., max_queue_size > 0)`.

## Build and run
- Build with `make server` (or `sh ./build.sh`), linking `-lpthread -lmysqlclient`.
- This codebase assumes Linux APIs (`epoll`, `signal`, `socketpair`, `mmap`); Windows builds are not the target.
- Runtime flags are documented in `README.md`: `-p`, `-l`, `-m`, `-o`, `-s`, `-t`, `-c`, `-a`.

## Editing guidance
- Preserve the existing split between infrastructure modules (`webserver/`, `threadpool/`, `timer/`, `log/`, `CGImysql/`) and HTTP behavior in `http/`.
- When changing request routing, check both `http/http_conn.cpp` and the matching static assets in `root/`.
- When changing DB or concurrency code, trace both the connection pool and the thread-pool call sites before editing.

