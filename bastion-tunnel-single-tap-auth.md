# Walkthrough: Single-Tap Hardware-Key Auth Through a Jump Host

This covers the technique behind reaching an isolated appliance VM for
RDP through a bastion — with only **one** hardware security key touch,
not two. It's a small trick, but the reasoning behind it is a genuinely
reusable pattern for anything gated behind a jump host.

## The problem it solves

The isolated appliance (call it the "target") isn't directly reachable —
it only accepts connections from the bastion, and the bastion only
accepts connections gated by a hardware security key. The obvious way
to reach the target from a client machine is SSH's built-in `ProxyJump`:

```
ssh -J bastion target
```

This works. It also authenticates **twice** — once to the bastion, once
to the target — because `ProxyJump` tunnels a completely independent,
end-to-end SSH handshake from the client to the target *through* the
bastion. The bastion is just plumbing in this model; your own identity
is what's being checked at both ends. Two hops, two touches.

## The alternative: let the bastion vouch for you

Instead of tunneling your own connection through, have the **bastion
itself** make the second connection, as a remote command:

```bash
ssh bastion "ssh target"
```

Here, the client authenticates **once**, to the bastion, with the
hardware key. The inner `ssh target` then runs *on the bastion*, using
whatever identity the bastion itself has configured for reaching the
fleet — in this setup, a certificate signed by the same CA the target
already trusts. The client's hardware key is never involved in the
second hop at all; the bastion is vouching for an already-authenticated
session, not re-authenticating the same person a second time.

This is the actual mechanism, not a shortcut — `ProxyJump` and a nested
remote command are fundamentally different at the protocol level, not
just different syntax for the same thing.

## Applying it to an RDP tunnel

The appliance runs a desktop environment reachable over RDP, but RDP
itself has no concept of jump hosts — so the tunnel needs to be
established first, then RDP pointed at the local end of it.

```bash
ssh -L 3390:localhost:3390 bastion "ssh -N -L 3390:localhost:3390 target"
```

Breaking this down:
- The **outer** `-L 3390:localhost:3390` forwards a local port on the
  client to `localhost:3390` *as seen from the bastion*.
- The remote command run on the bastion is itself another `ssh -L`,
  forwarding *that* connection onward to the actual target's RDP port.
- `-N` on the inner command means "don't run an interactive shell, just
  hold the forward open."

One real gotcha here: the **outer** command must not also use `-N`.
`-N` and a remote command are mutually exclusive — the whole point of
the outer hop is to execute that inner `ssh` command, so it needs to be
allowed to run a command. Only the inner hop, which has nothing further
to do but hold a forward open, uses `-N`.

## Wrapping it in a launcher

The full flow — start the tunnel, wait for it to come up, launch the RDP
client, tear the tunnel down when the RDP window closes — is a good
candidate for a small script rather than typing three commands in order
every time. A PowerShell version, annotated:

```powershell
# Clean up any leftover tunnel from a previous failed run before
# starting a new one — a stale process silently holding the port open
# is a common source of "it connects but nothing happens" bugs.
Get-CimInstance Win32_Process -Filter "Name = 'ssh.exe'" | Where-Object {
    $_.CommandLine -match "3390:localhost:3390"
} | ForEach-Object {
    Stop-Process -Id $_.ProcessId -Force -ErrorAction SilentlyContinue
}

# Start the nested tunnel (see above) — note no -N on the outer hop.
$ssh = Start-Process -FilePath "ssh.exe" `
    -ArgumentList "-L 3390:localhost:3390 bastion `"ssh -N -L 3390:localhost:3390 target`"" `
    -WindowStyle Hidden -PassThru

# Wait for the local end of the tunnel to actually come up before
# launching RDP against it — a fixed sleep is fragile; poll instead.
$timeout = 30; $elapsed = 0; $ready = $false
while ($elapsed -lt $timeout) {
    if ($ssh.HasExited) { break }
    if ((Test-NetConnection 127.0.0.1 -Port 3390 -WarningAction SilentlyContinue).TcpTestSucceeded) {
        Start-Sleep -Seconds 2   # give the inner hop a moment to fully settle
        $ready = $true
        break
    }
    Start-Sleep -Seconds 1; $elapsed++
}

if ($ready) {
    Start-Process -FilePath "mstsc.exe" -ArgumentList "/v:127.0.0.1:3390" -Wait
}

if (-not $ssh.HasExited) { Stop-Process -Id $ssh.Id -Force }
```

## Mistakes made building this (worth knowing before you hit them)

**Orphaned inner processes.** If a run is killed or fails partway, the
*inner* `ssh` process — running on the bastion, not the client — can
keep running, silently holding the target port open on the bastion.
The next run's own inner connection then fails to bind ("address already
in use"), while stale traffic may still route through the leftover
process. Cleaning up local (client-side) leftover processes isn't
enough; the bastion-side process needs its own cleanup too:

```bash
ssh bastion "pkill -f 'L 3390:localhost:3390 target'"
```

**Testing the wrong address family.** `Test-NetConnection localhost`
can resolve to `::1` (IPv6) while the SSH forward only bound IPv4 — the
readiness check silently never succeeds even though the tunnel is fine.
Use `127.0.0.1` explicitly, both for the readiness poll and for the
`mstsc` target.

**A visible console flash on launch.** `-WindowStyle Hidden` on the SSH
process is enough to hide *that* window, but if the script itself is
launched as `powershell.exe -WindowStyle Hidden ...`, some Windows
configurations still show a brief console flash before the hidden style
takes effect. The reliable fix is a tiny VBScript wrapper that launches
PowerShell via `WScript.Shell.Run(..., 0, False)` — the `0` window-style
argument means no window is ever created in the first place, rather
than being created and then hidden:

```vbscript
Set objShell = CreateObject("WScript.Shell")
scriptDir = CreateObject("Scripting.FileSystemObject").GetParentFolderName(WScript.ScriptFullName)
psPath = scriptDir & "\launcher.ps1"
cmd = "powershell.exe -NoProfile -ExecutionPolicy Bypass -File """ & psPath & """"
objShell.Run cmd, 0, False
```

Point a desktop shortcut at the `.vbs` file (not the `.ps1` directly —
Windows treats a `.vbs` pinned to the taskbar oddly if targeted
directly rather than via a proper shortcut with its own icon).

## Why this is worth doing at all

The alternative — just accepting two hardware-key touches per session
via `ProxyJump` — isn't wrong, and in fact is used deliberately
elsewhere in this architecture as the slower, more explicit "break-glass"
path precisely *because* it re-authenticates the client's own identity
at both hops rather than trusting the bastion to vouch. The single-tap
pattern is the right choice for routine, frequent access where the
bastion is already the trusted intermediary; the double-tap `ProxyJump`
path is the right choice when you deliberately want that extra,
independent check — for example, as a fallback that still works even if
you don't fully trust the bastion's own certificate chain in that
moment. Which one to use is a real design decision, not just a
convenience tradeoff.
