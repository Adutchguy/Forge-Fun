# Forge-Fun

Forge-Fun is a privacy-focused build of [Stable Diffusion WebUI Forge](https://github.com/lllyasviel/stable-diffusion-webui-forge) (itself built on [AUTOMATIC1111's Stable Diffusion WebUI](https://github.com/AUTOMATIC1111/stable-diffusion-webui) and [Gradio](https://www.gradio.app/)). It keeps everything Forge does and adds one main feature: **Secure Presets**, passphrase-protected, encrypted prompt and settings presets for txt2img and img2img. It also turns off the places where Forge normally leaves your prompts in plaintext.

> The original Forge README is preserved in [`docs/README.upstream.md`](docs/README.upstream.md). Forge's own docs, news and discussions live in the [upstream project](https://github.com/lllyasviel/stable-diffusion-webui-forge).

## Contents

1. [What's different from stock Forge](#whats-different-from-stock-forge)
2. [Requirements](#requirements)
3. [Install and run](#install-and-run)
4. [Using Secure Presets](#using-secure-presets)
5. [Application waterfall](#application-waterfall)
   - [A. Startup waterfall](#a-startup-waterfall)
   - [B. Web server waterfall](#b-web-server-waterfall-webui_worker)
   - [C. Generation request waterfall](#c-generation-request-waterfall)
   - [D. Secure Presets waterfall](#d-secure-presets-waterfall)
6. [Privacy hardening](#privacy-hardening)
7. [Security model and limits](#security-model-and-limits)
8. [What is and isn't in the repo](#what-is-and-isnt-in-the-repo)
9. [Repo map](#repo-map)
10. [Staying current with upstream](#staying-current-with-upstream)
11. [Troubleshooting](#troubleshooting)
12. [License and credits](#license-and-credits)

---

## What's different from stock Forge

| Area | Change | Where |
| --- | --- | --- |
| New feature | **Secure Presets**: an accordion in the txt2img and img2img tabs to save, load, delete and re-key encrypted presets | `extensions-builtin/secure-presets/`, `modules/secure_store.py` |
| Encryption | AES-256-GCM, key derived from your passphrase with scrypt; nothing readable is written to disk | `modules/secure_store.py` |
| Image metadata | Infotext (prompt, settings) is **never** embedded in generated images, even if a config file or the API asks for it | `modules/options.py`, `modules/ui_tempdir.py` |
| Side files | `.txt` infotext files and `log.csv` are never written | `modules/options.py`, `modules/shared_options.py` |
| Last-generation memory | Removed: Forge normally writes your last prompt to `params.txt` and the paste button (↙️) reads it back. Both are gone | `modules/processing.py`, `modules/infotext_utils.py`, `modules/ui.py`, `modules/ui_toprow.py` |
| Cleanup | Leftover `params.txt` and `secure_presets/params.enc` are overwritten with zeros and deleted at startup, never read | `extensions-builtin/secure-presets/scripts/secure_presets.py` |
| Packaging | Added `pip-constraints.txt`; `cryptography` is installed on first launch by the extension's `install.py` | `pip-constraints.txt`, `extensions-builtin/secure-presets/install.py` |

**Behavior you will notice if you are used to stock Forge**

- Images do not carry their generation parameters, so PNG Info and "send to txt2img" from a saved file will not recover a prompt.
- The ↙️ paste-parameters button next to the prompt box is removed.
- The three options above show "(always off: Secure Presets)" in Settings and cannot be turned on.

## Requirements

- A GPU that stock Forge supports (NVIDIA recommended). Forge-Fun adds no extra hardware needs.
- **Python 3.10** (the launcher warns if you use another version; it is tested with 3.10.6).
- Git.
- Internet access on first launch: PyTorch (default `torch==2.3.1`, `torchvision==0.18.1` from the CUDA 12.1 index), Python requirements, helper repositories, and `cryptography`.

Overrides use the same environment variables as Forge, such as `TORCH_COMMAND`, `TORCH_INDEX_URL`, `REQS_FILE` and `COMMANDLINE_ARGS`.

## Install and run

```bash
git clone https://github.com/Adutchguy/Forge-Fun.git
cd Forge-Fun
```

- **Windows:** double-click or run `webui.bat`.
- **Linux / macOS:** run `./webui.sh` (macOS also reads `webui-macos-env.sh`).

On first launch the script creates a virtual environment, installs everything, and opens the UI in your browser (default `http://127.0.0.1:7860`). That first run can take a long time.

**Command-line options:** create `webui-user.bat` (Windows) or `webui-user.sh` (Linux/macOS) and set `COMMANDLINE_ARGS`. Those two files are git-ignored on purpose so your personal options never get committed. Example for `webui-user.bat`:

```bat
@echo off
set COMMANDLINE_ARGS=--cuda-malloc --cuda-stream
call webui.bat
```

**Models:** put checkpoints in `models/Stable-diffusion/`, VAEs in `models/VAE/`, and so on. Model files (`*.safetensors`, `*.ckpt`, `*.pth`) are git-ignored.

## Using Secure Presets

Open **Secure Presets 🔒** in the txt2img or img2img tab. The two tabs share one library and stay in sync.

1. **First time:** choose a passphrase of **at least 12 characters** and click *Create encrypted vault*. **There is no recovery. If you forget the passphrase, the presets are gone.**
2. **Each session:** enter the passphrase and click *Unlock*. Click *Lock* when done, which wipes the key from memory.
3. **Save:** type a name (1-80 characters; reusing a name overwrites it) and click *Save current prompts + settings as preset*.
4. **Load:** pick a preset and click *Load preset*. Every saved field that exists on the current tab is applied.
5. **Delete:** pick a preset, tick *Confirm delete*, click *Delete selected preset*.
6. **Change passphrase:** open the nested accordion; the whole vault is re-encrypted under a new salt and key.

**What a preset stores**

| Group | Fields |
| --- | --- |
| Both tabs | prompt, negative prompt, sampler, scheduler, steps, width, height, batch count, batch size, CFG scale, distilled CFG scale, seed, and the top-of-page **UI** choice (`sd` / `xl` / `flux` / `all`) |
| txt2img only | hires prompt and negative prompt, hires enable, upscaler, hires steps, denoise, scale, resize X/Y, hires CFG and distilled CFG, hires sampler and scheduler |
| img2img only | denoising strength, resize scale |

Presets do not store the checkpoint, LoRAs or ControlNet settings.

Your vault is the file `secure_presets/vault.json`, created next to the app. It is git-ignored; back it up yourself if you care about it.

---

## Application waterfall

This section traces what runs, in order, from the moment you start the app. File and function names are real; follow them in the code.

### A. Startup waterfall

```text
webui.bat / webui.sh
 │   sets up venv, reads webui.settings.bat / webui-user.*, COMMANDLINE_ARGS
 ▼
launch.py : main()
 │   1. --dump-sysinfo?  write sysinfo file and exit
 │   2. startup_timer.record("initial startup")
 │   3. prepare_environment()            (skipped by --skip-prepare-environment)
 │        ├─ check Python version, print version/commit
 │        ├─ install torch + torchvision if missing  (or --reinstall-torch)
 │        ├─ CUDA availability test                    (skip: --skip-torch-cuda-test)
 │        ├─ install clip / open_clip / xformers / ngrok when requested or missing
 │        ├─ git clone helper repos into repositories/
 │        │     (assets, huggingface_guess, BLIP)
 │        ├─ pip install -r requirements_versions.txt  if not already met
 │        └─ run_extensions_installers()               (skip: --skip-install)
 │              └─ runs every extension's install.py, including
 │                 extensions-builtin/secure-presets/install.py  → pip install cryptography
 │   4. --test-server / --forge-ref-a1111-home handling (dev only)
 │   5. start()
 ▼
launch_utils.start()
 │   import webui          ← the import itself runs the heavy initialization below
 ▼
webui.py (module level, in order)
 │   startup_timer.record("launcher")
 │   initialize_forge()                 modules_forge/initialization.py
 │        └─ adds packages_3rdparty/ to sys.path, warns about removed flags
 │           (--lowvram etc.), applies --gpu-device-id / --cuda-malloc,
 │           initializes the memory manager
 │   initialize.imports()               torch, gradio, paths, shared, misc
 │   initialize.check_versions()        (skip: --skip-version-check)
 │   initialize.initialize()
 │        ├─ torch / lightning / asyncio / TLS / SIGINT fixes
 │        ├─ configure_opts_onchange()
 │        ├─ sd_models.setup_model()
 │        ├─ codeformer + gfpgan setup
 │        └─ initialize_rest()
 │              ├─ set samplers
 │              ├─ list extensions → restore config state file
 │              ├─ list SD models, localizations
 │              ├─ scripts.load_scripts()      ← loads every extension's scripts/,
 │              │     including secure_presets.py, which registers its component
 │              │     capture hook and wipes any legacy params files
 │              ├─ load upscalers, VAE list, UNets, hypernetworks
 │              └─ initialize extra networks (LoRA etc.)
 ▼
back in start()
 │   webui.webui()  → starts webui_worker in a daemon thread       (see B)
 │   main_thread.loop()  → the main thread now only runs GPU tasks (see C)
 ▼
app is serving requests
```

Two threads matter after startup:

- **`webui_worker` thread:** builds the UI, runs the Gradio/FastAPI server, and handles stop/restart.
- **Main thread (`modules_forge/main_thread.py`):** a loop that pulls queued generation tasks one at a time. Keeping all heavy model work on one thread is what makes model moving fast in Forge.

### B. Web server waterfall (`webui_worker`)

```text
webui_worker()  (webui.py)  — runs inside `while 1:` so it can restart the UI
 │  1. clean temp dir if the setting is on
 │  2. script_callbacks.before_ui_callback()
 │  3. shared.demo = ui.create_ui()
 │        ├─ builds the top row (modules/ui_toprow.py), txt2img, img2img, extras,
 │        │  settings, extensions tabs
 │        ├─ every component created fires on_after_component → Secure Presets
 │        │  records the ones it needs by elem_id
 │        └─ each script's ui() is called → Secure Presets builds its accordion
 │  4. demo.queue(64)                    (skip: --no-gradio-queue)
 │  5. demo.launch(...)                  host, port, TLS, auth, allowed paths
 │  6. strip CORSMiddleware              (blocks hostile web pages from driving the app)
 │  7. initialize_util.setup_middleware(app)
 │  8. progress API, UI API, optional REST API (--api), extra-network pages
 │  9. script_callbacks.app_started_callback(demo, app)
 │ 10. print startup time summary
 │ 11. wait loop: poll shared.state for "stop" / "restart" every 5 s
 │
 ├─ "stop"    → demo.close(), exit loop
 └─ "restart" → demo.close(), app_reload_callback, script_unloaded_callback,
                initialize.initialize_rest(reload_script_modules=True), loop to step 1
```

On a UI restart the vault state is not lost: Secure Presets rebuilds its panels when the txt2img tab is constructed again, and the unlock state lives in `modules/secure_store.py`, not in the page.

### C. Generation request waterfall

What happens when you press **Generate** on txt2img (img2img is the same shape through `modules/img2img.py`).

```text
Browser: Generate button
 ▼
Gradio event → modules/call_queue.py : wrap_gradio_gpu_call
 │     registers the task, takes queue_lock, shared.state.begin()
 ▼
modules/txt2img.py : txt2img()
 │     main_thread.run_and_wait_result(txt2img_function, ...)
 ▼                     (task handed to the main thread loop)
modules_forge/main_thread.py : loop()  → Task.work()
 ▼
txt2img_function → txt2img_create_processing(...)   builds the processing object `p`
 │     (prompt, sampler, size, CFG, hires, scripts, override settings)
 ▼
modules/processing.py : process_images(p) → process_images_inner(p)
 │     per batch:
 │       ├─ run script hooks (process, process_batch, ...)
 │       ├─ encode prompts, sample, decode
 │       ├─ post-process, hires pass if enabled
 │       └─ [removed in Forge-Fun] write last-generation params.txt
 ▼
modules/images.py : save_image(...)
 │     [Forge-Fun] enable_pnginfo is forced False  → no infotext in file metadata
 │     [Forge-Fun] save_txt is forced False        → no .txt next to the image
 ▼
result returns to the UI; Gradio serves previews through modules/ui_tempdir.py
      [Forge-Fun] the "parameters" metadata key is dropped when temp files are made
```

### D. Secure Presets waterfall

**Building the panel (during UI creation, see B step 3)**

```text
script_callbacks.on_after_component(_capture)
 │     for each Gradio component, if its elem_id is in _WANTED and it is a real
 │     input component (not a layout), store it in _components
 ▼
SecurePresetsScript.ui(is_img2img)       one call per tab; txt2img resets shared state
 │     ├─ pair preset keys with captured components (missing ones are logged and skipped)
 │     ├─ pick the panel state: no vault / locked / unlocked / cryptography missing
 │     ├─ build the accordion: setup, unlock, and main groups
 │     └─ wire buttons; link the txt2img and img2img panels so lock state and the
 │        preset list refresh in both without a page reload
```

**Vault lifecycle (`modules/secure_store.py`)**

```text
create_vault(passphrase)
 │   ≥ 12 chars → new header {v:1, kdf:scrypt, n=2^17, r=8, p=1, salt:16 random bytes}
 │   key = scrypt(passphrase, salt) (32 bytes)
 │   plaintext = {"presets": {...}}
 │   blob = nonce(12 random bytes) + AES-256-GCM(plaintext, aad = "vault-v1|" + header)
 │   atomic write: vault.json.tmp → fsync → rename over vault.json
 ▼
unlock(passphrase)   re-derive key, decrypt; wrong passphrase or tampering → error
 │   key is kept in memory only (a bytearray)
 ▼
save_preset / delete_preset / change_passphrase   re-encrypt and rewrite atomically
 ▼
lock()   overwrites the key bytes with zeros and drops the reference
```

Because the header is bound into the encryption as authenticated data, editing the file's KDF settings makes decryption fail instead of silently weakening anything.

**Loading a preset (two steps on purpose)**

```text
Load preset clicked
 ▼
Step 1  do_load_prepare
 │     reads the preset, stashes its values, and switches the UI radio
 │     (sd / xl / flux / all) if the preset used a different one
 │     — Forge's own handler for that radio resets size, CFG and sampler
 ▼
(browser waits ~800 ms so Forge's handler finishes)
 ▼
Step 2  do_load_apply
       writes every stored value into its component, then shows "Loaded preset"
```

Doing it in two steps is what keeps the radio switch from overwriting the values you just loaded.

---

## Privacy hardening

| Setting | Stock Forge default | Forge-Fun |
| --- | --- | --- |
| `enable_pnginfo` (infotext in PNG/JPG/WebP/AVIF metadata, stealth infotext) | on | **forced off** |
| `save_txt` (`.txt` infotext next to each image) | off | **forced off** |
| `save_write_log_csv` (`log.csv` from the Save button) | on | **forced off** |
| `params.txt` (last generation) | written every run | **removed**, legacy copies wiped |
| ↙️ paste-from-last-generation | present | **removed** |

"Forced off" is enforced in `Options.__getattr__` (`modules/options.py`, `forced_values`), so it holds regardless of `config.json`, the Settings page, or API calls that try to set them.

## Security model and limits

Secure Presets protects the **stored presets** and removes common plaintext leaks. It is not a complete privacy system. Know what it does not cover:

- **Unlocked means readable.** While the vault is unlocked the key is in process memory. Anyone with access to the running app or machine can use it. Lock when you step away.
- **The browser sees your prompts.** Prompt text lives in the page and in your browser's own form history or extensions.
- **Generated images are not encrypted.** They are saved to `outputs/` as normal image files. They just no longer carry the prompt inside them.
- **Other extensions are not audited.** A third-party extension can log or save prompts itself.
- **Console output.** Secure Presets avoids printing text on errors, but other components of Forge may print generation details to the terminal.
- **Network exposure.** If you use `--listen` or `--share`, protect the UI with `--gradio-auth` and TLS. The passphrase is typed into the web page.
- **Secure deletion is best effort.** Legacy files are zero-overwritten before deletion, which does not guarantee erasure on SSDs, journaling or cloud-synced folders.
- **No recovery.** There is no backdoor and no reset. Lose the passphrase and the vault is unreadable by design.

## What is and isn't in the repo

Git-ignored on purpose, so each user's data stays on their machine:

| Path | Why |
| --- | --- |
| `/secure_presets/` | your encrypted vault |
| `/config.json`, `/ui-config.json` | your settings and UI state |
| `/outputs`, `/output` | generated images |
| `/models/**/*`, `*.safetensors`, `*.ckpt`, `*.pth` | model files |
| `/embeddings`, `/extensions`, `/repositories`, `/venv`, `/tmp`, `/cache`, `/config_states/` | local installs and caches |
| `webui-user.bat`, `webui-user.sh` | personal launch options |
| `/DxDiag.txt` | local hardware report |

A fresh clone therefore starts with default settings and an empty vault.

## Repo map

```text
launch.py                      entry point: environment prep, then start()
webui.py                       initialization on import; webui_worker server loop
webui.bat / webui.sh           wrapper scripts (venv, then launch.py)
modules/
  launch_utils.py              prepare_environment(), installers, start()
  initialize.py                imports(), initialize(), initialize_rest()
  options.py                   Options class (forced_values lives here)
  shared_options.py            Settings definitions
  secure_store.py              encrypted vault: crypto + preset storage
  ui.py, ui_toprow.py          main UI construction
  ui_tempdir.py                Gradio temp files (drops "parameters" metadata)
  txt2img.py, img2img.py       request entry points
  processing.py                the generation pipeline
  images.py                    saving images
  call_queue.py                job queue and GPU lock
  scripts.py, script_callbacks.py   extension script loading and hooks
modules_forge/
  initialization.py            Forge bootstrap
  main_thread.py               single-thread GPU task loop
  main_entry.py                Forge UI controls (checkpoint, VAE, UI preset radio)
extensions-builtin/
  secure-presets/              Secure Presets (install.py + scripts/secure_presets.py)
  sd_forge_*, forge_*          bundled Forge extensions (ControlNet, LoRA, IP-Adapter, ...)
docs/README.upstream.md        original Forge README
```

## Staying current with upstream

Add the original Forge as a second remote and merge from it when you want new features:

```bash
git remote add upstream https://github.com/lllyasviel/stable-diffusion-webui-forge.git
git fetch upstream
git merge upstream/main
```

Expect conflicts only in the files Forge-Fun edits: `modules/options.py`, `modules/shared_options.py`, `modules/processing.py`, `modules/infotext_utils.py`, `modules/ui.py`, `modules/ui_toprow.py`, `modules/ui_tempdir.py`. After any merge, verify that the three forced options are still forced, that no new code writes `params.txt`, and that the Secure Presets element IDs still exist (the startup log says "fields not found in this UI version, skipped" when one has moved).

## Troubleshooting

| Symptom | Fix |
| --- | --- |
| Panel says "The 'cryptography' package is missing" | Restart the app so `install.py` can install it, or run `pip install cryptography` in the venv. |
| "Wrong passphrase, or the file was modified" | Re-enter the passphrase. If it is right, `secure_presets/vault.json` was edited or corrupted; restore your backup. |
| "Fields not found in this UI version, skipped" in the console | An element ID moved in a Forge update; edit the lists at the top of `secure_presets.py`. |
| Loading a preset keeps the old size, CFG or sampler | The UI radio switch is still finishing; load the preset again. |
| Torch or CUDA install fails | Set `TORCH_COMMAND` / `TORCH_INDEX_URL` for your GPU, or see the upstream README for GPU-specific help. |
| Port already in use | Add `--port 7861` (or another) to `COMMANDLINE_ARGS`. |

## License and credits

Released under the **GNU AGPL v3**, the same license as the projects it builds on. See [`LICENSE.txt`](LICENSE.txt).

Forge-Fun stands on the work of [lllyasviel's Stable Diffusion WebUI Forge](https://github.com/lllyasviel/stable-diffusion-webui-forge), [AUTOMATIC1111's Stable Diffusion WebUI](https://github.com/AUTOMATIC1111/stable-diffusion-webui), [Gradio](https://www.gradio.app/), and every extension and model author bundled or referenced there. If you run a modified version as a network service, the AGPL requires you to offer its source to your users.
