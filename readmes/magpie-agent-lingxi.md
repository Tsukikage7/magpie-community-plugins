# magpie-agent-lingxi

WPS 灵犀 (Lingxi) on magpie. magpie's models appear in 灵犀's 「自定义模型」 as
`magpie/<model>`, served through magpie's gateway and routing groups.

> **Unofficial.** Not affiliated with Kingsoft / WPS. Windows only.

## What it does

- A local proxy on `127.0.0.1` (3426, or the next free port) forwards
  `magpie/<model>` to magpie with the model's own name, so 灵犀's traffic keeps
  working across magpie's routing and model changes.
- The plugin's page (**插件 → WPS 灵犀**) has a **同步** button. Pressing it
  writes magpie's model list into 灵犀's own custom-model list.
- Models from the `wps-lingxi/*` provider are excluded, so 灵犀 does not call
  itself through magpie (which would loop and spend credits twice).

## How the sync works, and what it touches

灵犀 keeps its custom models in its **page's localStorage** — there is no
config file and no import for them. The only way to write them is the Chrome
DevTools Protocol, and CDP only exists in a 灵犀 started with
`--remote-debugging-port`. A sync is therefore:

1. Press **同步**, then confirm.
2. The plugin asks 灵犀's windows to close (WM_CLOSE, as 灵犀's own Quit does).
   **It never force-kills 灵犀.** If a window stays (it is asking something) or
   灵犀 stays in the tray, the sync stops and says so — quit 灵犀 yourself, then
   press 同步 again.
3. It starts 灵犀 once with `--remote-debugging-port`, writes the list, and
   refreshes the page (the open conversation is kept).
4. It closes 灵犀 again and starts it the way you normally start it — its
   Run / sign-in entry when it has one (a checkbox on the page turns that
   off), else a plain start. **The debug port is gone.**

The plugin never edits shortcuts or Run entries. After a successful sync the
model list is in 灵犀's own storage; 灵犀 started normally uses it.

## What a sync is needed for

- The **proxy** keeps `magpie/<model>` working through magpie's routing, so
  conversation works whenever magpie is running — 灵犀 can be started normally.
- A sync is needed when the **set** of models changes (a provider added or
  removed, a model list edited in magpie). Press 同步 again: one close and
  reopen. Switching which model you use in magpie does not need one.

## Lifetime

The proxy and the sync run **inside magpie's plugin host**. Turning the plugin
off, updating it, or quitting magpie ends them. Nothing the plugin starts
outlives magpie.

## Upgrade from 0.1.x on Windows

0.1.x added `--remote-debugging-port=9222` to the Lingxi shortcuts and Run
value it found and kept their original arguments in
`%USERPROFILE%\.config\magpie\lingxi-agent\shortcuts.json`. 0.2.0 does not
use them. To put them back:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File tools\restore-local.ps1          # what it would change
powershell -NoProfile -ExecutionPolicy Bypass -File tools\restore-local.ps1 -Apply   # change it
```

A Lingxi that is running with the port (started before the restore) keeps it
until it is quit and started again.

## Requirements

- Windows 10/11.
- magpie with agent-plugin support (0.1.x).
- WPS 灵犀 installed. It is found from the Windows uninstall registry or common
  install paths; `LINGXI_EXE` / `LINGXI_HOME` override that.
- No Node install: magpie's Bun runs it.

## Install

- From magpie → 插件 → 发现: search `magpie-agent-lingxi` and install.
- Or: `magpie plugin add github:dongv88/magpie-agent-lingxi`

## Use

1. In magpie: **Agent → WPS 灵犀 → switch it on, pick a model.** This writes
   the model to `~/.config/magpie/lingxi.json`.
2. Open **插件 → WPS 灵犀**, press **同步** (the first time; and again whenever
   the model list changes).
3. In 灵犀: **选择其他模型 → 自定义模型 → `magpie/<model>`**.

## Data and privacy

- The proxy and the sync listen on `127.0.0.1` only.
- The page writes only 灵犀's `claw_custom_model_configs` (the custom-model
  list) and its selected model (`cowork_model_option_v2`).
- It does **not** read or store your WPS credentials or conversation. Requests
  carry magpie's per-agent gateway key (`~/.config/magpie/lingxi.json`), the
  same one magpie writes for this agent.
- While 灵犀 runs with the debug port (only during a sync), any local process
  could control its renderer; that window is a few seconds per sync.
- State files: `%USERPROFILE%\.config\magpie\lingxi-agent` (`install.json`,
  `settings.json`, `host.log`).

## Troubleshooting

- **Agent missing**: restart magpie once; check that 灵犀 is installed.
- **“Lingxi closed its window but is still in the tray”**: right-click the tray
  icon → Quit, then 同步 again.
- **“still running. It may be asking something in its window”**: answer 灵犀's
  dialog (Save?), then 同步 again.
- **Models show but fail**: magpie must be running (the proxy is what serves
  them). `host.log` tells the proxy's port.
- **After a 灵犀 update**: 灵犀 may drop its storage; 同步 again.
- **Nothing syncs**: check `~/.config/magpie/lingxi-agent/host.log`.

## Disclaimer

Unofficial integration. "WPS" and "WPS 灵犀" are trademarks of Kingsoft. Use at
your own risk.

## License

MIT
