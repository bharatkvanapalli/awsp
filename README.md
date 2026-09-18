# awsp

Switch the **global** default AWS profile with one short command, so every terminal,
script and IDE hits the account you picked. Works with IAM Identity Center (SSO),
assume-role and access-key profiles, and never copies a credential anywhere.

```console
$ awsp

default -> acme-dev
  123456789012  arn:aws:sts::123456789012:assumed-role/AWSReservedSSO_AdministratorAccess_1a2b/you

profiles
    root       acme-management          keys
    personal   my-own-account           keys
  * dev        acme-dev                 sso      123456789012
    qa         acme-qa                  sso      210987654321
    prod       acme-prod                sso      345678901234

usage: awsp <profile> | awsp --login | awsp --clear

$ awsp qa
default -> acme-qa  sso 210987654321
  210987654321  arn:aws:sts::210987654321:assumed-role/AWSReservedSSO_AdministratorAccess_3c4d/you
```

- **New machine, or no profiles yet?** Start with [SETUP.md](SETUP.md). It covers adding
  an account both ways: Identity Center, and access keys with or without a session token.
- **Profiles already set up?** Install below, then `awsp <name>`.

## Install

Copy all five lines. They work from any directory, and the last one reloads your PATH so
`awsp` is found in the terminal you are already in.

```sh
git clone https://github.com/bharatkvanapalli/awsp.git ~/awsp
mkdir -p ~/bin
install -m 755 ~/awsp/awsp ~/bin/awsp
grep -q 'HOME/bin' ~/.zshrc || echo 'export PATH="$HOME/bin:$PATH"' >> ~/.zshrc
exec zsh
```

Check it with `awsp --help`. If that says `command not found`, `~/bin` is not on your
PATH: run `echo $PATH | tr ':' '\n' | grep bin` to see. On bash, use `~/.bashrc` in place
of `~/.zshrc` and `exec bash` at the end.

Requires the AWS CLI v2 (2.9 or newer, for `aws configure export-credentials`) and
python3, which macOS already has. `git clone` fails if `~/awsp` already exists; either
pull there instead, or clone somewhere else and adjust the `install` line.

## Commands

| Command | Does |
|---|---|
| `awsp` | Show the current default, its live identity, and every profile with its alias, kind and account id |
| `awsp <profile>` | Make that profile the default; an alias or a unique substring is enough |
| `awsp -i`, `awsp --login` | Run `aws sso login` for the current default's session |
| `awsp -l`, `awsp --list` | Profile names only, one per line |
| `awsp -c`, `awsp --clear` | Remove `[default]` so every command needs `--profile` again |

Name resolution goes: exact profile name, then alias, then a unique substring. If a
substring matches several profiles, a single SSO one wins.

## Short names

When a profile's real name is long, or does not contain the word you reach for, give it
an alias once:

```sh
aws configure set awsp_alias prod --profile company-platform-production-sso
aws configure set awsp_alias personal --profile my-own-account
```

`awsp prod` then picks that profile, whatever it is called. Aliases show in the first
column of `awsp`.

## How it works

Picking a profile rewrites only the `[default]` section of `~/.aws/config` to defer to
that profile:

```ini
[default]
# written by awsp: mirrors profile 'acme-qa'. No credentials are copied.
credential_process = aws configure export-credentials --profile acme-qa --format process
region = us-east-1
output = json
```

Every AWS SDK and tool understands `credential_process`, so the switch is global and
survives new terminals and reboots. Because it defers rather than copies:

- **SSO profiles work.** Credentials are fetched per command and expire on their own.
- **No secret is ever written.** Nothing lands in `~/.aws/credentials`.
- **Named profiles are untouched.** Only `[default]` changes, and the config file is
  backed up to `config.bak-awsp` before every write.

## Why not `export AWS_PROFILE`?

That only affects the terminal you ran it in. A new tab, a task launched from your
editor, or Terraform started by an IDE is back on the old account. `awsp` changes the
file all of them read, so there is one answer to "which account am I in".

Two things still beat the default, and `awsp` warns about both: an `AWS_PROFILE`
exported in your shell, and a `[default]` section holding credentials in
`~/.aws/credentials`.

## Caveats

- A tool that ships its own AWS SDK must be able to run the `aws` binary, since
  `credential_process` shells out to it.
- With an expired SSO sign-in, commands fail until `awsp --login`. That is the point:
  the credentials are short-lived.
- Repos that pin a profile per environment, for example Terraform wrappers that pass
  `--profile` explicitly, ignore the default by design.

## License

MIT, see [LICENSE](LICENSE).
