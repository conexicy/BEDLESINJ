# BedlesInj

A GUI tool that sets Fast Flags in the Roblox player.

**How it works.** The client keeps every flag it knows about in a list inside its own memory,
with the value for each one held in a small slot next to it. BedlesInj finds that list in the
running client and writes the values straight into it. The client then reads them like any
other value — there is no interception, no certificate, and nothing on disk is touched.

That means the flags it sets are not limited to the ones Roblox publishes in its allowlist.
Anything the client knows about can be set, including the ones Roblox's own settings filter
throws away.

**Read this before you use it.** Writing into another program's memory is something Roblox's
anti-cheat is allowed to look at. The handle this tool holds on the client is visible to
anything inspecting the process list, and a modified client can be told apart from an
untouched one. The tool is meant for use on your own account, and using it against a game you
share with others is against Roblox's terms. That is a decision for you to make, not a
technical detail — the mechanics are described plainly below so there are no surprises.

---

## Requirements

**To run the shipped build:** 64-bit Windows 10 / 11. Nothing else — no .NET install, no
administrator rights.

**To build from source:** the [.NET 10 SDK](https://dotnet.microsoft.com/download).

---

## Building the single-file release

```powershell
cd RbxFFlagInjector
.\build.ps1
```

`build.ps1` publishes `publish\BedlesInj.exe` — **one self-contained file** (~50 MB) that is
the entire product: copy it somewhere and double-click it.

For a normal development build without publishing:

```powershell
dotnet build -c Release
```

---

## Usage

1. Run `BedlesInj.exe`. There is nothing to install and nothing to approve. Flags never need a
   certificate or a proxy. No system setting is changed.

   The app puts itself in the notification area, so closing the window does not stop it working -
   click its icon there to open it again, or use the icon's menu to exit properly.

2. Add flags in the grid. Each row has a name, a value, and a **type**:
   * `bool`   – sets `true` / `false`
   * `number` – sets `240`
   * `string` – sets text (only a few flags hold text; the app tells you if one cannot)
3. Press **Apply flags**. The tool attaches to the running Roblox client, writes the values,
   and reads them back to confirm each one took.
4. Press **Launch Roblox** if Roblox is not already running. With **Apply on Roblox start**
   ticked, the flags are written while the client starts — which is what you want for the
   flags the client reads only once, such as the size of the render target.

The sections are at the **bottom** of the window: the mark on the left is the flag injector and
the gear on the right is the settings.

### Keeping the flags in place

Roblox refreshes its own flags from its servers every couple of minutes and overwrites
whatever was written. With **Keep re-applying** ticked (every *n* seconds, 30 by default), the
tool writes them again for as long as the client runs. Turn it off to write them once and
leave them.

### Optional: the launch URI

If you want to jump straight into a game, click **Play** on the Roblox website, copy the
`roblox-player:1+launchmode:play+...` link, and paste it into the **Launch URI** box. The
player is launched directly by the tool so the bootstrapper cannot overwrite anything.

### Uncapping the frame rate

The **Settings** tab has an **Uncap the frame rate** switch. Roblox holds the frame rate down
with a cap in the settings file the client keeps in its own data folder
(`%LOCALAPPDATA%\Roblox\GlobalBasicSettings_13.xml`), and the switch sets that cap to 9999.

Two things to know about it: the client reads the file when it starts, so **restart Roblox** for
it to take effect, and the client rewrites that file whenever you change a setting in game — which
puts the cap back. Turning the switch off restores the value that was there before.

### Finding the exact name of a flag

The names in the client's own registry do not always match the names in Roblox's published
settings — some flags have moved to Roblox's settings service and are not Fast Flags at all.
Press **Dump flags** and the tool writes the client's own list, with every flag's current
value, to `%LOCALAPPDATA%\BedlesInj\client-flags.json`. That file is the answer to "what is
this flag called, and what is it set to right now?".

### Command line

The executable has a few headless modes, useful for troubleshooting or for verifying a build
you are about to ship:

| Command | Effect |
| --- | --- |
| `--memorytest` | Attaches to the running client, finds its flag registry, reads a few flags, writes one and reads it back, then puts the original value back. |
| `--dumpflags` | Writes the client's whole flag list, with current values, to `client-flags.json`. |
| `--traytest` | Checks that closing the window leaves the app running in the notification area, and that the icon's Exit really ends it. |
| `--navtest` | Checks that the bar along the bottom moves between the sections. |
| `--makeicon <path>` | Writes the executable's icon, which Windows reads out of the file itself. |
| `--selftest` | Verifies the settings codec against the live CDN. Does not modify the machine. |
| `--themetest`, `--checktest`, `--scrolltest`, `--flagtest`, `--filetest`, `--addflagstest`, `--dialogtest` | The window's own checks. |

Each of these writes its report to `%LOCALAPPDATA%\BedlesInj\last-command.log` as well as to
the console, so the output is available even when launched without one.

---

## How it works

### Finding the flags

Roblox's client builds a list of fast variables while it starts. Each entry holds the flag's
name and points at a small object that points again at the value — a 32-bit slot in the
client's own data section.

Nothing about that list is published, so it is found at run time rather than hard-coded:

1. The client's writable sections are searched for a pointer to something that begins with a
   float `1.0`, which is how the registry's structure starts.
2. Each candidate is checked by walking its list and reading the names in it. A real registry
   gives back flag-shaped names; anything else gives back nonsense.
3. The lists with the most flags in them are read, and the values behind each flag are
   recorded: the name, the address of the value, and whether it holds a number or a switch.

On the current build this finds **15,172 flags** in about three seconds. The names are the
client's own, without the type prefix: `FFlagDebugDisplayFPS` is stored as
`DebugDisplayFPS`. A name that is typed with or without its prefix is found either way.

The result is kept in `flag-slots.json`, keyed by the client's version, so the search is only
done once for each Roblox build: the first run after an update takes a few seconds, and every run
after that is instant.

### Writing the values

Writing a value is a four-byte write into the client's own data section, followed by a read
back to confirm it took. If the read back does not match, the flag is reported as refused
rather than quietly left alone.

Two things are worth knowing:

- **The client overwrites its own flags.** It refreshes them from Roblox every couple of
  minutes, which is what the re-apply setting is for.
- **A few flags are readouts, not settings.** The flag that reports the size of the render
  target, for one, is written by the engine itself, so a value written to it is put straight
  back. The tool reads the value back and reports what actually happened.

### What it does not do

- It does not intercept anything. No traffic is read, no proxy is configured, and the system
  proxy setting is never touched.
- It does not modify the client executable, and it does not rewrite the client's settings except
  for the frame-rate cap switch.
- It does not need administrator rights. It does need to run as the same Windows account as
  Roblox, because that is what decides whether a process may write to another's memory.

---

## Files

Everything the tool writes lives in `%LOCALAPPDATA%\BedlesInj\`:

| File | Purpose |
| --- | --- |
| `flags.json` | Your saved flag profile. |
| `client-flags.json` | The client's own flag list, written by **Dump flags**. |
| `flag-slots.json` | Where each flag's value lives in the client, kept per Roblox version so the tool does not have to look for it again. |
| `injector.log` | The activity log. |
| `last-command.log` | Output of the most recent command-line run. |
| `settings.json` | The window's own preferences. |

The only file touched outside that folder is Roblox's own `GlobalBasicSettings_13.xml`, and only
when the frame-rate cap switch is used.

The executable itself is self-contained and can live anywhere, including a read-only location
or a temporary folder.

---

## Verifying the setup

With Roblox running, check that the tool can reach it and write a flag:

```powershell
.\BedlesInj.exe --memorytest
```

It attaches to the client, finds its flag registry, reads a few flags, writes one, reads it
back and puts the original value back. The report is also written to `last-command.log`.

---

## Troubleshooting

**"No running Roblox client to attach to."**
Roblox has to be running. The tool finds the flag list while the client starts, so if the
client has just been launched, wait a second or two and press **Apply flags** again.

**A flag is reported as "this client has no flag with that name".**
The client does not have that flag. Some names in Roblox's published settings belong to
Roblox's settings service rather than to the fast flags, and some flags have been renamed.
Press **Dump flags** and look the name up in `client-flags.json`.

**A flag is reported as "the client holds a switch here, not a number".**
The type in the grid does not match the type the client holds. Change the row's type.

**The flag worked and then stopped working.**
The client refreshed its own flags and put the old value back. Tick **Keep re-applying**.

---

## Notes on scope

Fast Flags are client-side configuration. This tool does not modify game servers, does not
read or alter network traffic, and does not patch any Roblox executable. Flags still behave
exactly as the client implements them — including any that Roblox has chosen to ignore, and
including any that the client reads only at startup.
