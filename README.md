# VS Code (Cirrus) — OnDemand app

An OpenOnDemand `batch_connect` app that launches a VS Code session, served by
[code-server], in the **Cirrus** Kubernetes cluster with the user's home
directory mounted.

This is the VS Code counterpart to the Cirrus Jupyter apps and follows the same
shape — `submit.yml.erb` renders a pod spec, init containers from
[`ood-k8s-utils`] mint a per-session password, and `view.html.erb` posts it to
log the user in. The differences from the Jupyter apps are deliberate and are
explained below.

[code-server]: https://github.com/coder/code-server
[`ood-k8s-utils`]: https://github.com/OSC/ood-k8s-utils

## `/rnode/`, not `/node/`

This is the one structural difference from the Jupyter apps, and it is not
interchangeable. OnDemand ships two proxies:

| Proxy | What the backend receives | Backend must |
| --- | --- | --- |
| `/node/HOST/PORT/x` | `/node/HOST/PORT/x` — prefix **kept** | know its own prefix |
| `/rnode/HOST/PORT/x` | `/x` — prefix **stripped** | assume it is at `/` |

Jupyter takes the first route, which is why the Jupyter app needs a third init
container that runs `kubectl` to discover its node name and NodePort just to
build `c.ServerApp.base_url`.

code-server has no base-path setting to compute. It emits relative roots and
assumes it is mounted at `/` — the rendered page references
`./_static/...`, `"serverBasePath":"."`, `"rootEndpoint":"."`. That is exactly
what `/rnode/` provides, so **this app needs no base-url plumbing at all** and
the third init container disappears. Pointed at `/node/` instead, code-server
would be asked for paths it does not serve and the session would come up blank.

## How the password works

1. `init-secret` generates a random password and stores it in `<pod>-secret`.
2. OnDemand's kubernetes adapter reads that Secret back and exposes every key in
   it to `view.html.erb` — that is where the `password` local comes from. The
   password is never rendered into an app template.
3. `add-passwd-to-cfg` copies it into the pod's ConfigMap, because the Secret is
   not mounted and the main container has neither `kubectl` nor the RBAC to
   fetch it.
4. `launch.sh` reads it out and exports `$PASSWORD`, which is what code-server's
   `--auth password` consumes.

**`launch.sh` waits for step 3 rather than assuming it.** Volumes are mounted
before init containers run, so `csenv` is already projected — empty — when the
init container edits the underlying ConfigMap. Kubelet watches ConfigMaps and
re-projects within a second or two, but that is a race, and losing it silently
would produce a session whose Connect button is rejected with no clue why. The
wait turns a race into a bounded, self-describing failure.

Because the port stays closed during that wait, the app also raises the startup
probe budget to five minutes; ood_core's default of ~27s would kill the pod
mid-wait.

### `$PASSWORD` beats a stale config file

code-server writes `~/.config/code-server/config.yaml` itself on first run, with
a random password of its own. That file lives on the NFS home and outlives the
session, so every session after the first finds one on disk with the *wrong*
password. Verified against code-server 4.131.0: the environment wins, and the
on-disk password is rejected. That is why this app sets `$PASSWORD` and does not
pass `--config`.

## Running as the real user

The pod's `securityContext` is the user's own UID and GID, which no stock image
knows about — a Jupyter-derived image's `/etc/passwd` was built around `jovyan`
(UID 1000). `/etc/passwd` and `/etc/group` are therefore supplied from the
ConfigMap with an entry for the real user, mounted with `subPath` so they replace
those two files and nothing else in `/etc`. Without it, `getpwuid()` fails and
every VS Code terminal opens with an `I have no name!` prompt, with git and ssh
quietly broken behind it.

`$SHELL` is pinned to `/bin/bash` rather than copied from the user's HPC passwd
entry: a login shell of `/bin/tcsh` is common here and almost never present in a
container image, and a shell that does not exist breaks terminals just as
thoroughly as no passwd entry at all.

## Choosing an image

The image needs a `code-server` executable and must be reachable from the Cirrus
cluster. **Most Jupyter images do not have one** — code-server is a separate
install, not part of the Jupyter Docker Stacks. The Cirrus JupyterHub base image
does, via the `code-server.dev` installer, which is why it is the default here.

`launch.sh` probes `/usr/bin`, `/usr/local/bin`, `/usr/lib/code-server/bin`,
`/opt/code-server/bin`, `$HOME/.local/bin`, and then `$PATH`, and if it finds
nothing it says so and lists where it looked — rather than failing with the bare
`executable file not found` that naming a path directly would produce.

## Persistence

`--user-data-dir` and `--extensions-dir` point at `$HOME/.local/share/code-server`
on the NFS home, so installed extensions, settings, and window layout survive the
pod.

If that directory cannot be created, the session does **not** die — it falls back
to container-local storage and logs a loud warning, because a session you can
open a terminal in is worth more than no session when you are trying to work out
why your home did not mount. Note that the fallback must also redirect
`XDG_DATA_HOME`: code-server resolves its own log directory through XDG
independently of `--user-data-dir`, and without that it retries the mkdir from a
background process and floods the log with uncaught `EACCES`.

## Form input is validated at the door

`submit.yml.erb` output is parsed by ood_core and re-rendered through *its*
`pod.yml.erb`, which interpolates values either bare (the image field) or inside
plain double quotes (env values). That template is in another gem and cannot be
fixed from here, so a form value containing `": "`, a double quote, `#`, or a
newline could restructure the generated Pod manifest. Both free-text fields are
therefore checked against a strict pattern and rejected with an explanatory error
before they reach any YAML. The same applies to GECOS, which is free text from
the site user database and would otherwise be able to inject a `/etc/passwd`
line.

The same constraint binds anything *this app* writes into a `command:` array,
not just user input. ood_core re-emits each element as `- "<element>"`, so a
literal double quote in a command closes the YAML string early and the whole
manifest dies at submit time with:

```
error converting YAML to JSON: yaml: did not find expected key
```

That is what the Jupyter app's `Q=$(printf '\047')` incantation is for — it
builds a quote at runtime so the literal never appears in the template. This app
sidesteps it by needing no quoting at all: the only interpolated value is the
password, which `create_passwd` builds from `[a-zA-Z0-9]` and which therefore
always expands to exactly one shell word. **Anything added to a command array
later must keep that property** — put it in `launch.sh` instead, where it lives
inside a block scalar and quoting is free.

## Files

| File | Purpose |
| --- | --- |
| `manifest.yml` | App name, category, and description shown in OnDemand |
| `form.yml` | Launch form (image, working directory, CPUs, memory, wall time) |
| `submit.yml.erb` | Pod spec, NFS mount, ConfigMap (incl. `launch.sh`), init containers |
| `view.html.erb` | Connect button; posts the session password to `/rnode/.../login` |
| `template/` | `batch_connect` template directory |

## Deploy as a dev app

**Develop → My Sandbox Apps → New App**, give it a directory name and this
repo's **HTTPS** git URL. App files live at the repo root so OnDemand finds
`manifest.yml`.

## Not yet done

- **Not yet run on Cirrus.** Everything below was verified locally (see next
  section); the pod itself has not been submitted through OnDemand.
- No GPU option. `gpus_per_node` is supported by ood_core but the resource name
  and node availability on Cirrus have not been confirmed here.
- Only the home directory is mounted, matching the Jupyter apps.
- `--log debug` is on deliberately while the app is new; worth dialing back once
  it has some mileage.

## What was verified locally

Rendered `submit.yml.erb` through ERB with a stubbed `OodSupport::User` and
checked the result parses as YAML with the fields ood_core expects, that the
command survives `Shellwords.split`, and that no two ConfigMap files declare the
same `mountPath` (which the API server rejects outright). Six injection attempts
against the two free-text fields and against GECOS were rejected.

`launch.sh` was then run against a real `codercom/code-server` container to
confirm: the password wait blocks the port and releases ~2s after the value
lands; every flag is accepted; `$PASSWORD` beats a pre-seeded on-disk
`config.yaml`; the session cookie is `code-server-session`; the served page uses
relative roots; and an unwritable home degrades to a warning plus a working
session instead of a flood of uncaught exceptions.
