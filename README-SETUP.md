# How this is set up on my PC

Set up 2026-10-12. Two reverse engineering tools are connected to Claude Code,
and they do different jobs. This file says which one to use, how to use it,
and how to update or remove it.

| Name in Claude Code | What it is | Use it for |
|---|---|---|
| `ghidra` | GhidraMCP 2.0 (LaurieWired). A plugin inside the Ghidra window plus a small Python server | Roblox, and anything I already have a Ghidra project for |
| `rea` | REA 6.4.0 (this repo's npm release). Runs its own hidden Ghidra in the background | A quick look at a file without opening Ghidra. .NET programs, Electron/JS apps, websites, HAR captures |

Both are registered for every project (Claude Code user config), so they work
from any folder.

## Which one to use

**`ghidra` (GhidraMCP)** works on the program open in my Ghidra window, with
all the analysis, renames and labels already in the project. It can also make
changes: rename functions, variables and data, add comments, set function
prototypes and variable types. Use this for Roblox.

**`rea`** never touches my Ghidra window or projects. Each session it copies
the file, imports and analyses it from scratch in a hidden Ghidra, answers
questions, then deletes everything. On Windows it can only read, not rename
or comment. Fine for small files, far too slow for a Roblox dump (it would
re-analyse the whole thing every session).

## Using `ghidra`

1. Start Ghidra: `Desktop\Desktop\Ghidra\ghidraRun.bat`.
2. Open the project and open the program in the CodeBrowser.
3. First time only: `File > Configure > Developer`, tick **GhidraMCPPlugin**.
4. Ask Claude, for example: "decompile the function at 0x1401234 in Ghidra",
   "list xrefs to this string", "rename FUN_14012 to ReadHumanoid".

Notes:

* It listens on `127.0.0.1:8080` only, and refuses requests from browsers or
  other machines. Change the port in `Edit > Tool Options > GhidraMCP HTTP Server`.
  If you change it, update the `--ghidra-server` URL in the Claude Code
  registration too.
* Only one CodeBrowser window can hold the port. A second window logs
  "Could not start the GhidraMCP backend".
* If Ghidra isn't open, Claude's Ghidra tools answer with a connection error.
  That's expected.

## Using `rea`

In Claude Code, name the file: "use rea to open C:\path\to\file.exe and find
where it reads the config". For a .NET program, an Electron app folder or a
`.har` file, just give the path; rea picks the right tool.

Things to know:

* It is read-only on Windows, and works on 32 and 64-bit native EXEs and DLLs
  on a local NTFS drive.
* The first call on a file waits for Ghidra's full analysis. A 220 KB test
  EXE took 49 seconds, mostly Ghidra starting up. A big binary can take many
  minutes.
* It never runs the file. It analyses a private copy.

From a terminal (PowerShell), set the same variables the Claude Code
registration uses, then call it:

```powershell
$env:REA_ANALYSIS_PROVIDER = 'ghidra'
$env:GHIDRA_INSTALL_DIR = "$env:USERPROFILE\Desktop\Desktop\Ghidra"
$env:JAVA_HOME = 'C:\Program Files\Java\jdk-21.0.10'
rea doctor --provider ghidra --json        # health check
rea inspect C:\path\to\file.exe --provider ghidra --format json
```

## Privacy settings applied

The code was audited before install: no malware, no telemetry, nothing about
me or my usage is sent anywhere. Three things did contact npm or GitHub by
default, and all three are switched off:

* `NO_UPDATE_NOTIFIER=1` is set as a Windows user variable. It stops the daily
  npm version check the CLI framework does.
* rea is installed globally and registered as `node ...\rea.mjs mcp`, not
  `npx -y rea-agents`, so starting Claude Code doesn't contact npm.
* Installed with `--omit=optional`, so the Frida package (which downloads a
  binary from GitHub) is left out. Never point Frida at Roblox anyway:
  Hyperion would likely catch it.

I also checked that all 143 npm packages have valid registry signatures, and
that the Windows native addon's SHA-256 matches its manifest.

GhidraMCP 2.0 was built from source on this PC (the release zip only targets
Ghidra 12.1.4, and I have 12.1.2). Its Python side only ever talks to Ghidra
on localhost.

## Updating

**rea.** Don't use `rea update`. Read the new release's changes first, then
use:

```powershell
npm install --global rea-agents@<version> --omit=optional
```

**GhidraMCP.** Rebuild it whenever Ghidra is upgraded, because Ghidra only
loads an extension built for its exact version:

```powershell
git clone https://github.com/LaurieWired/GhidraMCP
$env:GHIDRA_INSTALL_DIR = "$env:USERPROFILE\Desktop\Desktop\Ghidra"
& "$env:GHIDRA_INSTALL_DIR\support\gradle\gradlew.bat" -p GhidraMCP\extension buildExtension
# then in Ghidra: File > Install Extensions > + > GhidraMCP\extension\dist\<zip>, restart Ghidra
& "$env:LOCALAPPDATA\ghidra-mcp\venv\Scripts\python.exe" -m pip install --upgrade .\GhidraMCP
```

## Where everything is

| What | Where |
|---|---|
| rea | `%APPDATA%\npm\node_modules\rea-agents` |
| rea skill for Claude | `%USERPROFILE%\.claude\skills\reverse-engineer-anything` |
| GhidraMCP plugin | `%APPDATA%\ghidra\ghidra_12.1.2_PUBLIC\Extensions\GhidraMCP` |
| GhidraMCP Python server | `%LOCALAPPDATA%\ghidra-mcp\venv` (`Scripts\ghidra-mcp.exe`) |
| Old GhidraMCP plugin (backup) | `%APPDATA%\ghidra\GhidraMCP-old-backup-2026-10-12` |
| Both registrations | `%USERPROFILE%\.claude.json`, under `mcpServers` |

`claude` isn't on PATH on this PC. The CLI ships with the VS Code extension:
`%USERPROFILE%\.vscode\extensions\anthropic.claude-code-<version>-win32-x64\resources\native-binary\claude.exe`.

## Removing

```powershell
claude mcp remove rea -s user
claude mcp remove ghidra -s user
npm uninstall --global rea-agents
Remove-Item -Recurse "$env:USERPROFILE\.claude\skills\reverse-engineer-anything"
Remove-Item -Recurse "$env:LOCALAPPDATA\ghidra-mcp"
```

Then in Ghidra, `File > Install Extensions`, untick GhidraMCP and restart.

## About this fork

The setup above uses the npm release, not this fork's source. The npm package
includes the prebuilt Windows native addon, which this repo doesn't contain.
The fork is kept as a reference copy.
