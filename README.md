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
knows about — the code-server image's `/etc/passwd` was built around `coder`
(UID 1000). `/etc/passwd` and `/etc/group` are therefore supplied from the
ConfigMap with an entry for the real user, mounted with `subPath` so they replace
those two files and nothing else in `/etc`. Without it, `getpwuid()` fails and
every VS Code terminal opens with an `I have no name!` prompt, with git and ssh
quietly broken behind it.

`$SHELL` is pinned to `/bin/bash` rather than copied from the user's HPC passwd
entry: a login shell of `/bin/tcsh` is common here and almost never present in a
container image, and a shell that does not exist breaks terminals just as
thoroughly as no passwd entry at all.

`/etc/group` is seeded from **this host's** system range (GID < 1000) rather than
hand-written. Mounting the file at all hides the names the image shipped, and an
unnamed GID is not merely untidy — `groups` prints

```
groups: cannot find name for group ID 39
```

into every terminal, and the user has no way to tell that from a real fault. The
case that shows up in practice is a GPU session, where the runtime attaches the
group owning `/dev/nvidia*` so the container can open the device. That group is
doing its job; it just has no name once this file is mounted. The names can't be
hardcoded and can't be read from the image (not pulled anywhere at submit time),
but they don't need to be: supplementary GIDs are assigned by the **node**, not
the image, and this host shares the node's numbering. Only the system range is
copied — above 1000 is user and LDAP territory, and not ours to publish into a
container. The file is read directly rather than through `Etc.group`, which would
walk all of NSS, LDAP included.

## The image is fixed

There is no image field on the form. Which container VS Code runs in is not a
decision a user opening an editor should have to make, and getting it wrong
produces a session that dies on startup — code-server is a separate install, so
most images, including nearly every Jupyter image, simply do not have it.

The app runs upstream's own image, pinned:

```
docker.io/codercom/code-server:4.131.0
```

A blank slate — Debian 13, glibc 2.41, `code-server` at `/usr/bin/code-server`,
plus `bash`, `git`, `curl`, and `nano`. Deliberately **no** Python, compilers, or
scientific stack: users bring their own toolchain from their home directory,
which is mounted. If a session needs a preinstalled environment instead, change
the `editor_img` constant at the top of `submit.yml.erb` — that is the single
point of control, and the only requirement on a replacement is a `code-server`
executable and a POSIX shell.

Pinned to an explicit version rather than `:latest` so a session that worked
yesterday works today. Pulled from Docker Hub, as the `ood-k8s-utils` init
containers already are; worth mirroring into `hub.k8s.ucar.edu` eventually for
pull speed and to drop the Docker Hub dependency, at which point only that one
constant changes.

`launch.sh` still probes `/usr/bin`, `/usr/local/bin`,
`/usr/lib/code-server/bin`, `/opt/code-server/bin`, `$HOME/.local/bin`, and
`$PATH` for the binary. That is redundant for the pinned image and kept on
purpose: swapping the constant is the intended way to change environments, and
the probe makes doing so fail with a message naming the paths it searched rather
than a bare `executable file not found` from the container runtime.

## GPUs

A GPU session is flagged on the session card by `info.md.erb`. The card
deliberately does *not* report the image the way the Jupyter apps do — that is
fixed here rather than chosen on the form, so it would be the same constant on
every session. The GPU request is the one launch choice that changes what the
session can do, and nothing else on the card reveals it. Nothing is printed for
CPU-only sessions, matching the Jupyter apps so the two read the same way side
by side.

The form's **GPUs** checkbox sets `gpus_per_node`, which ood_core renders into the
pod's resource limits *and* requests as `nvidia.com/gpu: 1`. The key comes from
`native.gpu_type`, which defaults to `nvidia.com/gpu`; set it in `submit.yml.erb`
if the cluster advertises GPUs under a different name.

Leaving the box clear omits the line entirely rather than emitting
`nvidia.com/gpu: 0`. ood_core guards the field with
`unless script.gpus_per_node.nil?` — a nil check, not a zero check — so the ERB
has to skip the key, not zero it. Verified: the rendered CPU and GPU pod
manifests differ by exactly the two `nvidia.com/gpu` lines and nothing else.

### The image has no CUDA toolkit

Requesting a GPU does not make the editor image able to use one. The pinned image
is a blank slate, so **bring your own GPU-enabled environment from your home
directory.** That works better than it sounds: the node's device plugin injects
the driver and `nvidia-smi`, and conda- or pip-installed PyTorch and TensorFlow
wheels bundle their own CUDA runtime, so they need only the driver. `nvidia-smi`
will see the card regardless of what is installed.

If sessions should instead come with CUDA preinstalled, point `editor_img` at a
CUDA-based image that also carries code-server. The Cirrus JupyterHub GPU images
qualify: `cirrus-jhub-images` installs code-server in its `base` stage, and
`gpu-nb`, `tf-nb`, and `torch-nb` all derive from it, so they carry it too —

```
hub.k8s.ucar.edu/cirrus-jhub/jhub-gpu-nb:<tag>
hub.k8s.ucar.edu/cirrus-jhub/jhub-torch-nb:<tag>
hub.k8s.ucar.edu/cirrus-jhub/jhub-tf-nb:<tag>
```

Those are multi-GB images, which is the trade for not asking the user to build an
environment. Note they are built for Jupyter and their `code-server` lands in
`/usr/bin` via the `code-server.dev` installer, which `launch.sh` already probes
for first.

Confirmed working on Cirrus against an NVIDIA A10 — see
[Confirmed on Cirrus](#confirmed-on-cirrus).

### If a GPU session sits in Pending

**ood_core's pod template emits `nodeSelector` but has no `tolerations` support at
all.** If the GPU nodes carry a taint — `nvidia.com/gpu=present:NoSchedule` is a
common one — this app cannot schedule onto them, and no change to
`submit.yml.erb` will fix it, because there is no field to set. Check with:

```bash
kubectl get nodes -o custom-columns=\
'NODE:.metadata.name,GPU:.status.allocatable.nvidia\.com/gpu,TAINTS:.spec.taints'
```

If the GPU nodes are tainted, the options are to remove the taint, admit these
pods with a mutating webhook, or carry a patched pod template. If they are merely
labelled, add the label under `native.node_selector` in `submit.yml.erb`:

```yaml
    node_selector:
      nvidia.com/gpu.present: "true"
```

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
`pod.yml.erb`, which interpolates values into bare double quotes. That template
is in another gem and cannot be fixed from here, so a form value containing
`": "`, a double quote, `#`, or a newline could restructure the generated Pod
manifest. The working-directory field — now the only free-text input, since
fixing the image removed the other one — is therefore checked against a strict
pattern and rejected with an explanatory error before it reaches any YAML. The
same applies to GECOS, which is free text from the site user database and would
otherwise be able to inject an `/etc/passwd` line.

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
| `form.yml` | Launch form (working directory, CPUs, memory, GPU, wall time) |
| `submit.yml.erb` | Pod spec, NFS mount, ConfigMap (incl. `launch.sh`), init containers |
| `view.html.erb` | Connect button; posts the session password to `/rnode/.../login` |
| `info.md.erb` | Session card body; flags a GPU session |
| `template/` | `batch_connect` template directory |

## Deploy as a dev app

**Develop → My Sandbox Apps → New App**, give it a directory name and this
repo's **HTTPS** git URL. App files live at the repo root so OnDemand finds
`manifest.yml`.

## Confirmed on Cirrus

A GPU session launched through OnDemand and came up working: the terminal
prompt reads `<user>@code-server-<id>` — so the `/etc/passwd` mount resolved the
real user rather than leaving an `I have no name!` prompt — and `nvidia-smi`
reported an **NVIDIA A10**, driver 580.159.04, 23 GB. So the GPU nodes are
reachable without `tolerations` on this cluster, and the concern below about
`Pending` did not materialise.

`nvcc` is absent, as expected and as documented above: the node injects the
driver, not the CUDA toolkit.

## Not yet done

- The image is not mirrored into `hub.k8s.ucar.edu`, so first pull comes from
  Docker Hub (~1 GB).
- Only the home directory is mounted, matching the Jupyter apps.
- `--log debug` is on deliberately while the app is new; worth dialing back once
  it has some mileage.

## What was verified locally

Rendered `submit.yml.erb` through ERB with a stubbed `OodSupport::User` and
checked the result parses as YAML with the fields ood_core expects, that the
command survives `Shellwords.split`, and that no two ConfigMap files declare the
same `mountPath` (which the API server rejects outright). Injection attempts
against the working-directory field and against GECOS were rejected.

Both the CPU and GPU paths were rendered all the way through ood_core's own
`pod.yml.erb` and checked: `nvidia.com/gpu` present in limits and requests when
requested and absent otherwise, `gpus` of `0`/`''`/`nil`/`'off'` never requesting
one, `/etc/group` free of duplicate or malformed entries with no LDAP-range GID
leaked, `/etc/passwd` lines all 7 fields with no duplicate UID, and no double
quote in any command element. Both manifests pass
`kubectl create --dry-run=client`.

`launch.sh` was then run against a real `codercom/code-server` container to
confirm: the password wait blocks the port and releases ~2s after the value
lands; every flag is accepted; `$PASSWORD` beats a pre-seeded on-disk
`config.yaml`; the session cookie is `code-server-session`; the served page uses
relative roots; and an unwritable home degrades to a warning plus a working
session instead of a flood of uncaught exceptions.
