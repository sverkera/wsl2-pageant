# How to use Pageant ssh agent with WSL2

There are several way to accomplish this task. This text is inspired by https://sxda.io/posts/sharing-ssh-agent-wsl/ which describes
how to utilize systemd and npiperelay to forward openssh agent from Windows to a WSL2 instance.

With modern versions of Putty named pipes are used to communicate between tools and Pageant ssh agent, but unlike openssh agent the 
named pipe has a random name. Hence a variant of the above method is needed as outlined in the steps below:

1) Download npiperelay.exe. The original repository is no longer maintained so download from forked repository at https://github.com/albertony/npiperelay instead.
2) Extract npiperelay.exe to somewhere in Windows filesystem, this is neccesary as otherwise it will not be able to execute in windows. The scripts below will assume it is installed as %APPDATA%\npiperelay\npiperelay.exe.
3) Create a symbolic link pointing to $APPDATA/npiperelay/npiperelay.exe in ~/.local/bin/npiperelay.exe
```shell
mkdir -p ~/.local/bin
ln -s `wslpath "$(powershell.exe -Command '[System.Environment]::GetEnvironmentVariable("APPDATA")')" | tr -d '\r'`/npiperelay/npiperelay.exe ~/.local/bin/npiperelay.exe
```
4) Create the systemd socket unit
```shell
mkdir -p ~/.config/systemd/user/
cat <<EOF > ~/.config/systemd/user/named-pipe-ssh-agent.socket
[Unit]
Description=SSH Agent provided by Windows pageant named pipe

[Socket]
ListenStream=%t/ssh/ssh-agent.sock
SocketMode=0600
DirectoryMode=0700
Accept=true

[Install]
WantedBy=sockets.target
EOF
```
5) Create the systemd template unit
```shell
mkdir -p ~/.config/systemd/user/
cat <<EOF > ~/.config/systemd/user/named-pipe-ssh-agent@.service
[Unit]
Description=Proxy to Windows Pageant SSH Agent
Requires=named-pipe-ssh-agent.socket

[Service]
Type=simple
EnvironmentFile=%h/.config/systemd/user/named-pipe-ssh-agent.env
ExecStart=%h/.local/bin/npiperelay.exe -p -l -ei -s '\$SSH_AGENT_PIPE'
StandardInput=socket

[Install]
WantedBy=default.target
EOF
```
6) The trick is to figure out the name of pageant named pipe. That is done in .profile and put in an environment file read by systemd. We also add setting SSH_AUTH_SOCK
```shell
cat <<EOF >> ~/.profile
export SSH_AUTH_SOCK=/run/user/\$UID/ssh/ssh-agent.sock
echo >.config/systemd/user/named-pipe-ssh-agent.env SSH_AGENT_PIPE="\$(wslpath "\$(powershell.exe -Command '[System.IO.Directory]::GetFiles("\\\\.\\\\pipe\\\\")')" 2>&1 | grep pageant | tr '\\\\' '/')"
EOF
```
7) Activate the changes and test the result by listing the available keys.
```shell
source ~/.profile
systemctl daemon-reload --user
systemctl enable --user --now named-pipe-ssh-agent.socket
systemctl status --user named-pipe-ssh-agent.socket
ssh-add -l
```
