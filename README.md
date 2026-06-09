# ssh-askpass-hyprland
This can help you if you use ssh-keys with(either/both):
- password encryption
- [hardware token as second factor ](https://wiki.archlinux.org/title/SSH_keys#FIDO/U2F) (touch only - **if you have device with PIN you might need to do some tweaking**)

And have following setup:
- **Hyprland**(wayland)
- optional: ssh-agent ran by the user systemd service installed by `openssh` package (arch)

**The problem: There's no prompt for the password input / touching the Ubikey(or inputting PIN). And the existing solutions e.g. (aur) `openssh-askpass` don't work.**

> AFAIK it's possible to make `lxqt-openssh-askpass` work, but it looks weird to me (not meant to be part of a hyprland setup)

## Steps
1. Create a tiny script for making the notification & password prompt at `~/.local/bin/ssh-askpass-hyprland`:
```
#!/bin/bash
# ssh hands an askpass program three different kinds of prompts; with
# SSH_ASKPASS_REQUIRE=force *all* of them come here (even from a terminal),
# so we have to recognise each type and answer it correctly.

prompt="$1"

case "$prompt" in
    *[Pp]assphrase*|*[Pp]assword*)
        # A real secret is being requested (password-encrypted or
        # verify-required key). Ask for it.
        zenity --password --title="$prompt" 2>/dev/null
        ;;
    *"(yes/no"*|*"(y/n)"*|*fingerprint*|*"continue connecting"*)
        # A confirmation, e.g. an unknown host key. ssh expects "yes"/"no".
        if zenity --question --title="SSH" --text="$prompt" 2>/dev/null; then
            echo "yes"
        else
            echo "no"
        fi
        ;;
    *)
        # No secret needed (e.g. FIDO/U2F security-key touch prompt).
        # Just nudge the user; printing nothing is the correct answer here.
        notify-send "🔑 SSH" "${prompt:-Touch your YubiKey}" -t 5000
        ;;
esac
```
> both `notify-send` and `zenity` should be already present on a hyprland setup

```
chmod +x ~/.local/bin/ssh-askpass-hyprland
```
2. Add overridde for the systemd service
```
systemctl --user edit ssh-agent
```
Setup the required env vars (this is what tells ssh-agent to use our notification script)
```
[Service]
Environment=SSH_ASKPASS=/home/strudelpi/.local/bin/ssh-askpass-hyprland
Environment=SSH_ASKPASS_REQUIRE=force
```
3. restart
```
systemctl --user daemon-reload
systemctl --user restart ssh-agent
```
4. your `~.bashrc` or similar:
```
export SSH_ASKPASS=/home/strudelpi/.local/bin/ssh-askpass-hyprland
export SSH_ASKPASS_REQUIRE=force
export SSH_AUTH_SOCK="$XDG_RUNTIME_DIR/ssh-agent.socket"
```
5. You should now have a working integration with ssh - when ssh (e.g. through `git` if you use ssh for git) will trigger the notification and prompt.

## Troubleshooting

### `Host key verification failed` / `read_passphrase: requested to askpass` when connecting to a new host

If you see something like this on a first connection to an unknown host:
```
/etc/ssh/ssh_known_hosts2 does not exist
debug1: read_passphrase: requested to askpass
Host key verification failed.
lost connection
```
this is **not** a passphrase problem. Because `SSH_ASKPASS_REQUIRE=force` routes *every* prompt to the askpass script (even from a terminal), the *"Are you sure you want to continue connecting (yes/no/[fingerprint])?"* host-key confirmation also goes to the script. An older script that only handled `passphrase` prompts returns an empty answer here, so ssh treats the host key as rejected.

The script above fixes this by detecting yes/no confirmations and answering them via a `zenity --question` dialog (it works regardless of whether a key uses an `sk`/FIDO token, a passphrase, or both). Unsetting `SSH_ASKPASS_REQUIRE` "fixes" it only because ssh then falls back to the terminal for the yes/no question — but that defeats the GUI/agent integration.
