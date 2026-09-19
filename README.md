# docker-setup

Install Docker Engine on Ubuntu or Debian, the official way, in one pass.

Docker's own documentation is a page of commands to copy one by one: add a
key, add a repository, install five packages, then remember the two things it
does not mention, capping container logs before one of them fills the disk
and letting your user run `docker` without `sudo`. This script does all of it,
asks before each choice, shows the plan, and checks the repository's signing
key against Docker's published fingerprint before installing anything.

```
$ docker-setup

System
  os            Ubuntu 24.04.2 LTS  (amd64)
  docker        not installed

Questions
  press enter to accept the value in brackets

  Members of the docker group run containers without sudo. That is the same power as root, so only someone you would give root to.
  User who runs docker without sudo [louis] (- for none):

  Without a cap, one chatty container fills the disk in a few weeks and nothing warns you. Each container keeps a few files of a fixed size.
  Cap container logs? [Y/n]
    Size of each log file [10m]:
    Files kept per container [3]:

  Pulls a tiny image, runs it once and removes it, to prove the engine works from end to end.
  Run a test container at the end? [Y/n]

To apply
  repository    Docker's own, for ubuntu noble, signing key checked
  engine        docker-ce with the cli, containerd, buildx and compose
  logs          10m x 3 files per container
  user          louis runs docker without sudo
  test          hello-world container, then removed

  Apply these 7 steps? [Y/n]

┌─ [1/7] Prerequisites  21:11:11
│  ✓ ca-certificates
│  ✓ curl
│  ✓ gnupg
└─ ✓ ready

┌─ [2/7] Signing key  21:11:17
│  fingerprint 9DC858229FC7DD38854AE2D88D81803C0EBFCD88
│  /etc/apt/keyrings/docker.asc
└─ ✓ matches Docker's published fingerprint

┌─ [3/7] Repository  21:11:17
│  /etc/apt/sources.list.d/docker.list
│  https://download.docker.com/linux/ubuntu noble stable, amd64
└─ ✓ declared

┌─ [4/7] Docker Engine  21:11:18
│  ✓ containerd.io
│  ✓ docker-ce-cli
│  ✓ docker-buildx-plugin
│  ✓ docker-compose-plugin
│  ✓ docker-ce
└─ ✓ 29.8.1, compose 5.5.1

┌─ [5/7] Log rotation  21:11:42
│  /etc/docker/daemon.json
└─ ✓ 10m x 3 files per container, applies to containers created from now on

┌─ [6/7] User access  21:11:44
│  louis added to the docker group
└─ ✓ applies to new sessions: log out and back in, or run: newgrp docker

┌─ [7/7] Test  21:11:45
└─ ✓ a container ran and was removed, the engine works

──────────────────────────────────────────────────────────────
Done in 35 s, 0 warnings
  docker        29.8.1
  compose       5.5.1
  service       active / enabled
  logs          10m x 3 files per container
  access        louis, without sudo
  log           /var/log/docker-setup.log

  ! log out and back in before using docker without sudo

  The docker group applies to new sessions only. This closes the current SSH session; reconnect and docker works without sudo.
  Log out now? [Y/n]
  logging out
```

While a step runs, a spinner says what is going on and apt prints one tick
per package configured, so nothing ever looks stuck. A step that could not do
everything ends with `!` instead of a tick, and the last line counts those.

## Install

```bash
git clone https://github.com/Lousblue/docker-setup.git
sudo install -m 755 docker-setup/docker-setup /usr/local/bin/
```

One bash script. It needs `sudo` or root, `apt-get`, and network access to
`download.docker.com`.

## Use

```bash
docker-setup                          # ask, showing what was detected
docker-setup --status                 # look, change nothing
docker-setup --yes                    # every default, no questions
docker-setup --user bob --log-size 50m --yes
docker-setup --no-user --no-test --yes
docker-setup --update                 # newer engine from Docker's repository, nothing else
```

Pass any option and it stops asking - which is also what happens without a
terminal, so it is safe to call from cloud-init or a provisioning tool. What
you did not mention takes its default.

### Settings

| Setting | Default | Option |
|---|---|---|
| User who runs `docker` without `sudo` | whoever ran sudo | `--user NAME`, `--no-user` |
| Cap on each container log file | `10m` | `--log-size SIZE`, `--no-log-rotation` |
| Log files kept per container | `3` | `--log-files N` |
| Run `hello-world` at the end | yes | `--no-test` |

On a machine that already has a cap, the values in place are the defaults,
not the built-in ones: running the script again never silently undoes a
choice made before.

### Other options

| | |
|---|---|
| `--status` | show what is installed, change nothing |
| `--update` | update the engine and its plugins from Docker's repository, touch nothing else |
| `--dry-run` | show what would be done, then stop |
| `--yes`, `-y` | never ask, take every default |

## What it does, exactly

1. **Prerequisites.** `ca-certificates`, `curl`, `gnupg`.
2. **Signing key.** Downloads `https://download.docker.com/linux/<distro>/gpg`
   to a temporary file, reads its fingerprint and compares it with the one
   Docker publishes, `9DC8 5822 9FC7 DD38 854A E2D8 8D81 803C 0EBF CD88`. A
   mismatch stops the script with nothing installed. A match goes to
   `/etc/apt/keyrings/docker.asc`.
3. **Repository.** One line in `/etc/apt/sources.list.d/docker.list`, pinned
   to that key, your architecture and your release codename.
4. **Docker Engine.** `docker-ce`, `docker-ce-cli`, `containerd.io`,
   `docker-buildx-plugin`, `docker-compose-plugin`. The service is enabled
   and started.
5. **Log rotation.** `/etc/docker/daemon.json` with the `json-file` driver
   and your cap. If a `daemon.json` without logging options is already there,
   it is copied to a dated `.bak` next to it and the step says so.
6. **User access.** `usermod -aG docker`. It takes effect at the next login.
7. **Test.** `docker run --rm hello-world`, then the image is removed.

## Files it writes

| | |
|---|---|
| `/etc/apt/keyrings/docker.asc` | Docker's signing key |
| `/etc/apt/sources.list.d/docker.list` | the repository |
| `/etc/docker/daemon.json` | log rotation |
| `/var/log/docker-setup.log` | everything apt, curl and docker had to say |

## Tested on

Ubuntu 24.04 and Debian 12 containers: engine installed, started, updated
and a `hello-world` container run, plus the interactive path and reruns on
a machine that already had everything.

## Caveats

- **The docker group is root.** Anyone in it can mount the host's disk into a
  container and read it. The script says so before asking, and `--no-user`
  keeps `sudo docker` the only way in.
- **Debian and Ubuntu only**, because those are the two Docker builds `.deb`
  packages for. Derivatives such as Mint or Raspberry Pi OS report their own
  name in `/etc/os-release` and are refused rather than guessed at.
- **Log rotation applies to containers created after it.** Existing
  containers keep the logging they were started with until they are
  recreated.

## Licence

MIT.
