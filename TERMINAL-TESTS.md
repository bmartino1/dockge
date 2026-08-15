# Dockge-PVE terminal test plan

Run these after building and deploying the fork.

## 1. Dockge Shell terminal identity

Open **Console → Dockge Shell** and run:

```bash
echo "TERM=$TERM"
stty size
printf 'cols='; tput cols
printf 'lines='; tput lines
clear
```

Expected:

- `TERM=xterm-256color`
- `tput` returns numeric rows/columns with no `unknown terminal` error
- `clear` works

## 2. Keyboard/Readline

Test:

- Up/Down command history
- Left/Right cursor movement
- Home/End
- Backspace/Delete
- `Ctrl+C`
- `Ctrl+L`
- TAB completion, for example `cd /data/Do<TAB>`

No key should be filtered by the browser-side line editor because that line editor has been removed.

## 3. Full-screen TUI programs

Run:

```bash
mc
nano /tmp/dockge-terminal-test.txt
vim /tmp/dockge-terminal-test.txt
```

Resize the browser while each application is open. The application should redraw to the new terminal size rather than duplicate panels or retain stale dimensions.

## 4. Clipboard

Test on Chrome, Edge and/or Brave:

1. Copy a one-line command from another page and press `Ctrl+V` in Dockge.
2. Paste with `Ctrl+Shift+V`.
3. Right-click inside the terminal and choose **Paste**.
4. Select terminal output and press `Ctrl+Shift+C`.
5. Select output and use right-click **Copy**.
6. Paste a multi-line shell block and confirm Bash bracketed-paste behavior is preserved.

Plain `Ctrl+C` must still send SIGINT to the terminal rather than acting as Copy.

## 5. PVE Host Shell

With the host target enabled, select **PVE Host Shell** and run:

```bash
whoami
hostname
uname -a
echo "TERM=$TERM"
stty size
printf 'cols='; tput cols
printf 'lines='; tput lines
```

Expected for the example PVE configuration:

- `whoami` is `root`
- `hostname` is the PVE host
- `TERM=xterm-256color`
- remote rows/columns match the browser terminal

Then test `mc`, `nano` or `vim` on the PVE host and resize the browser.

## 6. Reconnect/switch targets

Switch several times between **Dockge Shell** and **PVE Host Shell**. Confirm output from the inactive target does not appear in the active terminal and reopening a target restores its buffered screen/output.

## 7. Container terminal regression

Open a normal Dockge container terminal from a stack and verify:

- shell input works
- Ctrl keys work
- copy/paste works
- resize works

This verifies the shared `Terminal.vue` changes did not regress container exec sessions.
