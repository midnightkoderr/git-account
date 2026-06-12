# git-account

Manage multiple git identities (GitHub, GitLab, Bitbucket, self-hosted) on one machine.

> **Status:** working and tested.

## How it works

Each account gets an alias (e.g. `github-personal`, `gitlab-work`). Running `git-account use` inside a repo shows a numbered picker, then sets `user.name` / `user.email` locally and rewrites the remote URL to route through the right credentials. `git push` / `git pull` just work after that.

Supports two auth modes per account:

| Mode | How it works | Best for |
|---|---|---|
| **SSH** | ed25519 key per account, `Host` alias in `~/.ssh/config` | personal dev — no expiry, no rotation |
| **Token** | fine-grained PAT stored in system keyring via libsecret | CI accounts, scoped/expiring access |

## Install

```bash
git clone <this-repo>
sudo cp git-account /usr/local/bin/
# or without sudo:
cp git-account ~/.local/bin/
```

Token mode requires `libsecret-tools`:
```bash
sudo apt install libsecret-tools
```

## Quickstart

```bash
# 1. Register accounts (once per account)
git-account add

# 2. Activate an account in any repo
cd ~/projects/some-repo
git-account use
```

`git-account use` with no arguments shows a numbered picker:

```
Select account:

  #    ALIAS                  HOST               USERNAME             AUTH
  --   -----                  ----               --------             ----
  1)   github-personal        github.com         koderr               ssh
  2)   github-work            github.com         tootsy               ssh
  3)   gitlab-personal        gitlab.com         helium               token

Pick [1]:
```

## Commands

| Command | Description |
|---|---|
| `git-account add` | Register a new account (interactive) |
| `git-account list` | List all configured accounts |
| `git-account use [--global] [alias]` | Set account for current repo — picker if no alias given; `--global` sets as the global default |
| `git-account show` | Show identity + remotes for current repo |
| `git-account key <alias>` | Print SSH public key (SSH accounts only) |
| `git-account token set <alias>` | Store / update a PAT in the system keyring |
| `git-account token get <alias>` | Print the stored token |
| `git-account token clear <alias>` | Remove token from keyring |
| `git-account remove <alias>` | Remove an account |

## Global default

`--global` sets `user.name` / `user.email` in `~/.gitconfig` and wires credentials globally — useful before cloning, or to set a default identity for new repos.

```bash
git-account use --global github-personal
git clone git@github-personal:acme/api.git   # uses global identity automatically
```

**SSH accounts** — sets `core.sshCommand` globally so every repo using that key works without per-repo config:
```
core.sshCommand = ssh -i ~/.ssh/id_github-personal -o IdentitiesOnly=yes
```

**Token accounts** — registers a global credential helper scoped to the account's host:
```
credential.https://github.com.helper = !git-account cred github-work
```

Per-repo settings (from `git-account use` without `--global`) always take precedence over the global default.

## SSH mode

`add` generates an ed25519 key and adds a `Host` entry to `~/.ssh/config`:

```
Host github-personal
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_github-personal
  IdentitiesOnly yes
```

`use` rewrites the remote to use that alias:
```
https://github.com/acme/api.git  →  git@github-personal:acme/api.git
git@github.com:acme/api.git      →  git@github-personal:acme/api.git
```

After setup, add the public key to your provider once:
```bash
git-account key github-personal   # prints key + provider URL
ssh -T git@github-personal        # verify it works
```

## Token mode

`add` stores the PAT in the **system keyring** (GNOME Keyring / KWallet via libsecret) with the username saved as a lookup attribute — so you can find which account owns a token:

```bash
# stored attributes: service=git-account, account=<alias>, username=<username>
git-account token get gitlab-personal     # retrieve token
git-account token set gitlab-personal     # rotate/update token
```

`use` rewrites the remote to HTTPS and wires a per-repo credential helper that reads from the keyring automatically:

```
git@gitlab.com:acme/api.git  →  https://gitlab.com/acme/api.git
                                 credential.helper = !git-account cred gitlab-personal
```

`git push` calls the helper, which does `secret-tool lookup service git-account account gitlab-personal` and returns `username` + `password` to git — nothing manual.

## Fine-grained tokens and libsecret

`git-credential-libsecret` (the default helper) keys stored credentials by `protocol://hostname` only — so two GitHub accounts would share the same `https://github.com` key and overwrite each other. Token mode in this script avoids that by using its own namespaced keyring entries (`service=git-account, account=<alias>`) and a per-repo credential helper, so any number of accounts on the same host can coexist.

## Supported providers

| Provider | Hostname | SSH keys | Fine-grained tokens |
|---|---|---|---|
| GitHub | github.com | Settings → SSH keys | Settings → Tokens (beta) |
| GitLab | gitlab.com | Preferences → SSH Keys | Preferences → Access Tokens |
| Bitbucket | bitbucket.org | Account → SSH keys | Account → App passwords |
| Self-hosted | any hostname | varies | varies |
