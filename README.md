# clashrup

[![CI](https://github.com/spencerwooo/clashrup/actions/workflows/ci.yml/badge.svg)](https://github.com/spencerwooo/clashrup/actions/workflows/ci.yml)
[![Release](https://github.com/spencerwooo/clashrup/actions/workflows/release.yml/badge.svg)](https://github.com/spencerwooo/clashrup/actions/workflows/release.yml)
[![GitHub release (latest by date)](https://img.shields.io/github/v/release/spencerwooo/clashrup)](https://github.com/spencerwooo/clashrup/releases/latest)

Simple CLI to manage your systemd `clash.service` and config subscriptions on Linux.

- Setup, update, apply overrides, and manage via `systemctl`. **No more, no less.**
- Systemd configuration is created with reference from [*Running Clash as a service*](https://github.com/Dreamacro/clash/wiki/Running-Clash-as-a-service).
- No root privilege is required. `clash.service` is created under user systemd by default.
- `clashrup` got its name from [clashup](https://github.com/felinae98/clashup), a friendly Python alternative.

![clashrup setup and update](https://user-images.githubusercontent.com/32114380/214243049-00aa6c1d-1393-4e3d-b0f2-7ab648f0d27d.png)

## Installation

### Using a one-liner installation script

To install `clashrup`, inside your terminal, run:

```bash
curl -fsSL https://raw.githubusercontent.com/spencerwooo/clashrup/main/install.sh | sh -
```

By default, this installs the executable to `~/.local/bin/clashrup`. Add `~/.local/bin` to `$PATH` if needed.

### Downloading `clashrup` manually

Download prebuilt binary for Linux from [releases](https://github.com/spencerwooo/clashrup/releases/latest). Move under
`/usr/local/bin` (system-wide), `~/.local/bin` (user) or any other directory in your `$PATH`. Example:

```bash
curl -LO https://github.com/spencerwooo/clashrup/releases/download/{VERSION}/clashrup-{TARGET_ARCH}.tar.gz
tar -xvzf clashrup-{TARGET_ARCH}.tar.gz
mv clashrup ~/.local/bin/clashrup
```

### Building from source

Alternatively, clone the repo and install from source:

```bash
cargo install --path .
```

## Usage

> **Note**: Run `clashrup --help` to see a list of available commands.

To setup and start `clash` as a systemd service on a new Linux device, run:

```bash
clashrup setup
```

`clashrup` will first attempt to read from `~/.config/clashrup.toml` for config. If not found, it will generate a
default one and abort. You would then need to edit the config file and run `clashrup setup` again. See
[Configuration](#configuration) for more details.

With a valid config, rerun `clashrup setup` to download `clash` binary and remote config.

Ultimately, `clashrup setup` will attempt to:

- Download `clash` binary from `remote_clash_binary_url` and extract it to `clash_binary_path`.
- Download clash remote config from `remote_config_url`, apply overrides, and save it under `clash_config_root`.
- Create a user systemd service file `clash.service` under `user_systemd_root`.
- Enable and start `clash.service` with `systemctl`.

You can then check the status of the newly created `clash.service` running in the background with:

```bash
clashrup status
```

![clashrup status](https://user-images.githubusercontent.com/32114380/214243273-ac391d2b-8dcb-4ef0-ab14-ecc08aaa3f81.png)

If something doesn't work as expected, you can check the logs with:

```bash
clashrup log
```

![clashrup log](https://user-images.githubusercontent.com/32114380/214243360-12bf5b4e-7c71-4c13-ba16-1499bc11f7d9.png)

To update clash's config from remote and restart `clash.service`, run:

```bash
clashrup update
```

![clashrup update](https://user-images.githubusercontent.com/32114380/214244906-5281a4ea-09a3-4a74-aaf1-d14fcd2ca1b9.png)

If you modified config overrides in `~/.config/clashrup.toml`, you can apply them to clash's config
(`~/.config/clash/config.yaml`) and restart `clash.service` with:

```bash
clashrup apply
```

Finally, to stop `clash.service` and uninstall `clash` and config, run:

```bash
clashrup uninstall
```

Additionally, you would often need to set environment variables to proxy your traffic through `clash` within your
current terminal session. `clashrup` provides a convenient command to generate command for exporting environment
variables (`http_proxy`, `https_proxy`, and `all_proxy`) for your current session:

```bash
clashrup proxy export
```

> **Note**: For more proxy export commands, check `clashrup proxy --help`.

## Configuration

`clashrup` stores its config at `~/.config/clashrup.toml` by default.

Default config is generated upon setup (with command `setup`) as:

```toml
# ~/.config/clashrup.toml
remote_clash_binary_url = ""
remote_config_url = ""
remote_mmdb_url = "https://cdn.jsdelivr.net/gh/Dreamacro/maxmind-geoip@release/Country.mmdb"
clash_binary_path = "~/.local/bin/clash"
clash_config_root = "~/.config/clash"
user_systemd_root = "~/.config/systemd/user"

[clash_config]
port = 7890
socks_port = 7891
allow_lan = false
bind_address = "*"
mode = "rule"
log_level = "info"
ipv6 = false
external_controller = "127.0.0.1:9090"
# external_ui = "folder"
# secret = ""
```

where,

- Field `remote_clash_binary_url` should point to a downloadable gzipped `clash` binary URL
  ([example](https://github.com/MetaCubeX/Clash.Meta/releases/download/v1.14.0/Clash.Meta-linux-amd64-v1.14.0.gz)).
- Field `remote_config_url` should point to your subscription provider's config URL, which will be downloaded to
  `{clash_config_root}/config.yaml` during `setup` and `update`.
- Field `clash_config` holds a subset of supported config overrides for clash's `config.yaml`. Inside, `port`,
  `socks_port`, `mode`, and `log_level` are required. Other fields are optional. For a full list of configurable clash
  `config.yaml` fields, see [clash - Configuration](https://github.com/Dreamacro/clash/wiki/configuration).

If clash binary already exists at `clash_binary_path`, then `remote_clash_binary_url` will be ignored and `setup` will
skip downloading and setting up clash binary. (In this case, `remote_clash_binary_url` can be left empty.)

Other fields should be self-explanatory.

## Manage clash's settings

We recommend setting `external_controller` and `secret` for clash's RESTful API, which can be used to manage clash via
external dashboards like [yacd](https://github.com/haishanh/yacd).

If you are using this on a remote Linux server, edit `~/.config/clashrup.toml` and set `external_controller` to `:9090`:

```diff
-external_controller = "127.0.0.1:9090"
+external_controller = ":9090"
```

to allow external access. Run `clashrup apply` to apply the changes to clash and restart clash. You can now use
`http://{YOUR_SERVER_IP}:9090` to access the API and control clash's settings.

> **Warning**: Set `secret` if external access is granted to prevent unauthorized access to your clash API.

## Shell completions

`clashrup` has built-in support for generating shell completion information for the CLI. `clashrup` will output to
stdout completions with `clashrup completions <shell>`. `bash`, `zsh`, and `fish` are currently supported.

### bash

```bash
mkdir -p ~/.local/share/bash-completion/completions
clashrup completions bash > ~/.local/share/bash-completion/completions/clashrup
```

### zsh

```bash
mkdir -p ~/.local/share/zsh/site-functions
clashrup completions zsh > ~/.local/share/zsh/site-functions/_clashrup
```

Additionally, add directory `~/.local/share/zsh/site-functions` to your `fpath` where completions are loaded from.
Ideally, above where `compinit` is called in your `~/.zshrc`:

```diff
+fpath=(~/.local/share/zsh/site-functions $fpath)
 autoload -Uz compinit
 compinit -u
```

### fish

```bash
clashrup completions fish > ~/.config/fish/completions/clashrup.fish
```

## License

[MIT](LICENSE)
