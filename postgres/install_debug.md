# Pitfalls and Traps - Debugging Postgres on MacOS 

## Trap 1: when using macOS, compiling in a separate directory returns error

According to Postgres' official document:

> You can also run `configure` in a directory outside the source tree, and then build there, if you want to keep the build directory separate from the original source files. This procedure is called a *VPATH* build. 

But if you are using macOS, doing so causes following error when you run "make":

```bash
clang: error: no such file or directory: 'replication/repl_gram.o'
clang: error: no such file or directory: 'replication/repl_scanner.o'
clang: error: no such file or directory: 'replication/slot.o'
clang: error: no such file or directory: 'replication/slotfuncs.o'
clang: error: no such file or directory: 'replication/syncrep.o'
clang: error: no such file or directory: 'replication/syncrep_gram.o'
clang: error: no such file or directory: 'replication/syncrep_scanner.o'
clang: error: no such file or directory: 'replication/walreceiver.o'
clang: error: no such file or directory: 'replication/walreceiverfuncs.o'
clang: error: no such file or directory: 'replication/walsender.o'
clang: error: no such file or directory: 'utils/fmgrtab.o'
```

The only way out is to run `configure` directly in source tree. 

However, if you are using linux, there is no such bug and building in a separate file is completely okay.

## Trap 2: PG cannot be started by a root user

Postgres must be run by a non-root user. This causes great trouble when you try to use `VSCode + Container` for development: when you connect to the container, you login in as root user by default. Thus, if you "attach to a running container" in `vscode`,  you cannot start the server. 

The solution is to change the default user into a non-root user, such as `postgres`. This can be achieved by save the current running container as an image and start a container running it with `--user postgres`.

## Trap 3: cpptools fails on macOS-mounted directories

When we `attach to a running container` in vscode, it login in as the default user and create a directory named `.vscode-server` under the `$HOME` . If the `$HOME` is within a mounted volume, `cpptools` fails to function and returns "fails to connect" error.

```bash
cpptools client: couldn't create connection to server. Launching server using command /home/.vscode-server/extensions/ms-vscode.cpptools-1.28.3-linux-x64/bin/cpptools failed. Error: spawn /home/.vscode-server/extensions/ms-vscode.cpptools-1.28.3-linux-x64/bin/cpptools EACCES
```

The solution is to create a directory in container's own volume, grant the user rights and set it as the `$HOME`:

```bash
usermod -d /the_home_directory postgres
```

