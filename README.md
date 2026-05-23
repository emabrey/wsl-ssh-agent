## WSL2 compatibility

At the moment AF_UNIX interop does not seem to be working with WSL2 VMs. Hopefully this will be sorted out eventually. Meantime there is an easy workaround (proposed by multiple people) which does not use `wsl-ssh-agent.exe` and relies on combination of linux socat tool from your distribution and [npiperelay.exe](https://github.com/jstarks/npiperelay). *For example* put `npiperelay.exe` on drvfs for interop to work its magic (I have `winhome ⇒ /mnt/c/Users/rupor`, copy [wsl-ssh-agent-relay](docs/wsl-ssh-agent-relay) into your `~/.local/bin directory`, and add following 2 lines to your .bashrc/.zshrc file:

```bash
${HOME}/.local/bin/wsl-ssh-agent-relay start
export SSH_AUTH_SOCK=${HOME}/.ssh/wsl-ssh-agent.sock
```

**NOTE:** If you are having issues using `wsl-ssh-agent-relay` with systemd try adding `:WSLInterop:M::MZ::/init:PF` to `/usr/lib/binfmt.d/WSLInterop.conf`. For example (thanks to [rkl110](https://github.com/rkl110) - [Microsoft/WSL - Issue 8843](https://github.com/microsoft/WSL/issues/8843)):

```bash
sudo sh -c 'echo :WSLInterop:M::MZ::/init:PF > /usr/lib/binfmt.d/WSLInterop.conf'
```

Alternatively if you prefer to directly use systemd support in WSL2 /etc/wsl.conf:

```ini
[boot]
systemd = true
```

you could create wsl-ssh-agent.service unit in `/usr/lib/systemd/user`, something similar to:

```ini
[Unit]
Description=Windows SSH Agent Proxy via npiperelay
StartLimitIntervalSec=0

[Service]
Delegate=true
Type=exec
KillMode=process
ExecStart=/usr/bin/socat UNIX-LISTEN:'/run/user/<user id, usually 1000>/wsl-ssh-agent.sock',fork EXEC:'/home/<user name>/winhome/.wsl/npiperelay.exe -ei -s //./pipe/openssh-ssh-agent',nofork

[Install]
WantedBy=default.target
```

and then enable it:

```bash
systemctl --user daemon-reload
systemctl --user enable --now wsl-ssh-agent.service

export SSH_AUTH_SOCK=${XDG_RUNTIME_DIR}/wsl-ssh-agent.sock