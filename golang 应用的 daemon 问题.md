# golang 应用的 daemon 问题

## daemonize 面临的问题

- Goroutines complicate daemonization in process. `issues/227`
- Be aware that there are certain problems when the daemonization happens after goroutines are launched. `issues/227`
- You'd want to use whatever the target OS offers you...`upstart`, `systemd`, etc.
- For systems with `systemd` this is absolutely not necessary.
- Go daemon is not needed on systems with `systemd`.

## 比较

以下哪种方式是最合适的？

- `go run myapp.go &`
- `nohup go run myapp.go`
- `go run myapp.go & disown`
- use an external tool like `daemonize`: `daemonize -p /var/run/myapp.pid -l /var/lock/subsys/myapp -u nobody /path/to/myapp.exe`
- 基于 `systemd` 实现

## 项目

- [fiorix/go-daemon](https://github.com/fiorix/go-daemon) -- 基于 C 写的，又称 god
- [sevlyar/go-daemon](https://github.com/sevlyar/go-daemon) -- 基于 go 写的

## 参考

- [How to start a Go program as a daemon in Ubuntu?](https://stackoverflow.com/questions/10067295/how-to-start-a-go-program-as-a-daemon-in-ubuntu)
- [issues/227](https://github.com/golang/go/issues/227) -- a bug report regarding the ability to daemonize from within a Go program
- [The Problem with a Golang Daemon](http://www.ryanday.net/2012/09/04/the-problem-with-a-golang-daemon/)

