My team uses systemd. When systemd starts a process, it needs to know if the process started successfully. The most simple way to do this is for systemd to wait for a bit after starting the process (perhaps 10 seconds) and simply check that the process is still running. Since you generally can't predict exactly how long your service will take to successfully start up, this is error prone. Furthermore, this method is slow. If you are doing synchronous restarts for a rolling deploy, this adds a lot of time.

There is a better way called systemd-notify. Basically what happens is that your process uses IPC (inter-process communication) to tell systemd when it is up. Systemd provides a C library to accomplish this.

My team is running nodejs services, so we need to use systemd-notify from the node process. We have a few options to accomplish this, each with some minor drawbacks.

## Existing Solutions

### sd-notify npm package

This is probably your best option. This provides bindings to the systemd-notify C library. It requires a native build with node-gyp, but it is the most proper way. Using this approach, you not only can notify systemd when your app starts up, but also provide a heart beat so that systemd knows if your app becomes unresponsive without crashing and it will then know to recycle your app.

Drawback: Just that you have a dependency on node-gyp.

### Shell out to systemd-notify binary

systemd provides the `systemd-notify` binary which you can just shell out to. Simple, reasonable solution except that my team ran into a race-condition bug with it. Until that is fixed, I would recommend against this approach.

Drawback: race-condition bug

The details of the race-condition are as follows.

1. Your process creates a child process to execute `systemd-notify`
2. The `systemd-notify` process sends a message via IPC to systemd
3. The `systemd-notify` process exits
4. systemd (or sd-bus?) attempts to get the PID of the process that sent the message. It fails because the process has already exited.

The race condition is between steps 3 and 4. Sometimes step 4 completes before step 3 in which case everything works find, but it is nondeterministic.

To solve this, `systemd-notify` just needs to sleep for a tiny bit to give `sd-bus` a chance to get the PID. It is possible this problem has been fixed since we last investigated.

### Shell out to python

Python has a module that calls `systemd-notify` so you can write a little python one liner and shell out to it instead of `systemd-notify`. The reason this works is because you can add a short sleep to your python script to solve the race condition.

TODO: provide the python one liner here

## A Better Solution?

### TL;DR

I hoped to create a pure nodejs solution that didn't have a node-gyp solution, but I found that to be impossible due to nodejs' lack of support of the `unix_dgram` mode in their `dgram` module.

## Details

`systemd-notify` uses D-Bus to talk to `systemd`. Under the hood, that is just sending data to the AF_UNIX unix socket located at `/run/systemd/notify`. I wondered how complicated that message was and, if it was simple enough, if I could program nodejs to send raw data directly to that socket, thereby bypassing the node-gyp dependency.

The first step was to determine what D-Bus messages look like and what data systemd-notify was sending. Instead of trying to understand the D-Bus protocol and digging through systemd's code, we decided to just call `systemd-notify` and sniff the socket to see what data was sent.

To do this, we tried a man-in-the-middle approach as follows:

```
sudo mv /path/to/sock /path/to/sock.original
sudo socat -t100 -x -v UNIX-LISTEN:/path/to/sock,mode=777,reuseaddr,fork UNIX-CONNECT:/path/to/sock.original
```

I don't know why exactly, but this didn't work. Moving the socket around like this caused systemd-notify to fail. So we set about to find another way to sniff the data. `strace` to the rescue. We crafted a bare-bones python script that simply sleeps for 10 seconds (so we had time to grab its PID) and then called systemd-notify.

Before the 10 seconds ran out, we grabbed the PID for the file and started an `strace` on it like this: `strace -ff -o log -s 20000 [pid]`

Doing this we got the following output:

```
socket(AF_UNIX, SOCK_DGRAM|SOCK_CLOEXEC, 0) = 3
getsockopt(3, SOL_SOCKET, SO_SNDBUF, [212992], [4]) = 0
setsockopt(3, SOL_SOCKET, SO_SNDBUFFORCE, [8388608], 4) = 0
sendmsg(3, {msg_name={sa_family=AF_UNIX, sun_path="/run/systemd/notify"}, msg_namelen=21, msg_iov=[{iov_base="READY=1", iov_len=7}], msg_iovlen=1, msg_controllen=0, msg_flags=0}, MSG_NOSIGNAL) = 7
```

Here we can see the socket is opened and sent the message "READY=1". That's it! We were feeling hopeful. All we had to do was figure out how to open a unix domain socket and send the text "READY=1" to it.

Our first attempt was using `net.connect`. It is capable of connecting to unix domain sockets. However, we got the error EPROTOCOLERR (TODO: double check this error). To get a little more color on this, we decided to run `strace` on our nodejs script and compare it to the python script. Doing this, we found that python was opening the socket with `socket(AF_UNIX, SOCK_DGRAM|SOCK_CLOEXEC, 0) = 3` and node was opening it with `socket(AF_UNIX, SOCK_STREAM|SOCK_CLOEXEC|SOCK_NONBLOCK, 0) = 10`.

So node was using a stream mode of some sort and python was using a datagram mode.

After digging around a bit, we found that the proper way to do this in nodejs is to use the `dgram` module instead of the `net` module. It is similar, but supports datagram protocols like `udp4`, `udp6` and `unix_dgram`. Well, it used to support `unix_dgram`, but not anymore.

If you check the current node docs, you'll see that the docs only mention `udp4` and `udp6`. It turns out that the support for `unix_dgram` was removed in version 0.6 in 2011. You can see details in [this thread](https://groups.google.com/forum/#!topic/nodejs/iCzhcuxGP1I). I was also able to find [the `unix_dgram` option documented in an old nodejs branch](https://github.com/openwebos/nodejs/blob/master/doc/api/dgram.markdown).

If `unix_dgram` was still supported in node, I think it would be easy to implement the IPC to tell systemd when our process is up without a dependency on `node-gyp`, bug since it is not, I think your best option is to stick with the `sd-notify` npm package.
