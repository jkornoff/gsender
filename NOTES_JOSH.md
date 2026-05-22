# gSender -- Personal Build Notes

Forked from `Sienci-Labs/gsender` on 2026-05-22 to get the Height Map autoleveling tool
working today, ahead of upstream merge. Not maintained -- when upstream merges PR #798,
throw this away and use official releases.

## What's here

- **Branch:** `feature/heat-map-tool-support` (krudoy's PR #798 -- Height Map Tool)
- **Personal fix on top:** commit `5de55c156` -- "fix(heightmap): re-zero active WCS Z to lowest probed point"
- **PR link:** https://github.com/Sienci-Labs/gsender/pull/798

## The Z-zero fix

PR #798 normalizes the height map so the lowest probed point is Z=0 in the map. But
the machine's WCS Z=0 was never shifted to match, so transformed g-code cut shallow
by the depth of the lowest probe (see `hschnait`'s comment on the PR -- he was
re-probing the low point manually).

Fix lives in [src/app/src/features/HeightMap/index.tsx](src/app/src/features/HeightMap/index.tsx)
in `completeProbing`: after probing finishes, emit `G10 L2 P<n> Z<minProbeMPos>` to
shift the active WCS (G54..G59) Z origin to the lowest probed machine Z. That's the
re-touch-off, automated.

## Build the Windows installer

Prereqs: Node 24, yarn (`npm i -g yarn`), git bash. Use bash, not PowerShell.

```bash
cd /c/dev/gsender
yarn build              # ~50 sec
yarn build:windows      # ~2 min -> output/gSender-1.5.7-x64.exe
```

You'll see a `node-gyp / Visual Studio` error mid-build. **Ignore it.** The serialport
package ships N-API prebuilts (`node_modules/@serialport/bindings-cpp/prebuilds/win32-x64`),
which are ABI-stable and get bundled regardless. electron-rebuild only matters if those
prebuilds are missing.

## Dev mode (hot reload, in browser)

```bash
cd /c/dev/gsender
yarn dev                # http://localhost:8000
```

Backend (Node + serialport) runs on localhost; React UI served to browser. Same code
path as the Electron app -- Electron is just Chromium pointing at the same bundle.

## Install on the laptop

1. Copy `C:\dev\gsender\output\gSender-1.5.7-x64.exe` to the laptop.
2. Run it. NSIS installer, per-machine, needs admin once.
3. Open the app, connect to GRBL over USB.
4. Tools tile -> Height Map. Touch off Z somewhere on the stock first, then probe.
   The fix will re-zero Z to the lowest probed point automatically when probing completes.

## If upstream PR #798 is rebased/changed

```bash
cd /c/dev/gsender
git fetch origin pull/798/head:pr-798-new
git diff feature/heat-map-tool-support pr-798-new   # see what changed
# decide: rebase the fix onto pr-798-new, or just rebuild and re-apply the fix
```

## Where the fix touches code

Only file changed: [src/app/src/features/HeightMap/index.tsx](src/app/src/features/HeightMap/index.tsx)

- Added `activeWcs` selector reading `state.controller.modal.wcs`
- In `completeProbing`: before normalizing the map, compute `minProbeZ` from raw
  PRB machine-Z values, map active WCS to P-number, send `G10 L2 P<n> Z<minProbeZ>`
