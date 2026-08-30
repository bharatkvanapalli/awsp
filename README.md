# awsp

Switch the **global** default AWS CLI profile from one command.

```console
$ awsp

default -> work   123456789012  admin

profiles
  * work      123456789012  admin
    personal  210987654321  bharat

usage: awsp <profile> | awsp --clear

$ awsp personal
default -> personal   210987654321  bharat
```

## Why not just `export AWS_PROFILE`?

`export AWS_PROFILE=personal` only affects the terminal you ran it in. Open a
new tab, run a script from your editor, or launch Terraform from an IDE, and
you are back on the old account.

`awsp` rewrites the `[default]` section of `~/.aws/credentials`, so the switch
applies **everywhere** — every terminal, and every tool that reads the shared
credentials file: the AWS CLI, Terraform, CDK, Amplify, boto3, and the other
SDKs.

Your named profiles are never modified. Only `[default]` is rewritten.

## Install

Requires `bash`, `python3`, and the AWS CLI v2 — all present by default on
macOS and most Linux distributions.

```bash
mkdir -p ~/bin
curl -fsSL https://raw.githubusercontent.com/bharatkvanapalli/awsp/main/awsp -o ~/bin/awsp
chmod +x ~/bin/awsp
```

If `~/bin` is not already on your `PATH`:

```bash
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.zshrc   # or ~/.bashrc
source ~/.zshrc
```

## Usage

| Command | Effect |
|---|---|
| `awsp` | show the current default and list profiles |
| `awsp <profile>` | make `<profile>` the global default |
| `awsp -l`, `--list` | print profile names only |
| `awsp -c`, `--clear` | remove the default profile entirely |
| `awsp -h`, `--help` | usage |

### `--clear` is worth knowing about

With no default profile, a bare `aws s3 ls` fails with *"Unable to locate
credentials"* instead of quietly running against whichever account happened to
be default. If you work across several accounts and one of them is production,
that loud failure is usually what you want — it forces `--profile` to be
explicit.

## What it does about the ways this goes wrong

**Confirms where you landed.** Every switch ends by calling
`sts get-caller-identity` and printing the real account ID and IAM user. You
never have to trust that the switch worked, and broken credentials surface
immediately rather than on your next command.

**Warns when `AWS_PROFILE` shadows the default.** If that variable is set in
your shell, it silently overrides `[default]` — so `awsp` can appear to do
nothing. This is the most confusing failure mode there is, so `awsp` detects
it and tells you.

**Backs up before writing.** `~/.aws/credentials` is copied to
`~/.aws/credentials.bak-awsp` before any change.

**Never stores credentials of its own.** Keys are read from your existing
profiles at runtime and written only to `~/.aws/credentials`. Nothing is
cached, logged, or printed — the script contains no secrets, and neither does
this repository.

## How the active profile is detected

`[default]` is a copy, so it carries no record of where it came from. Rather
than keep a state file that can drift, `awsp` compares the access key ID in
`[default]` against each named profile and marks the one that matches. If you
edit `~/.aws/credentials` by hand, the display stays correct.

## Limitations

Only profiles with static access keys (`aws_access_key_id` /
`aws_secret_access_key`) can be made default. SSO profiles, `credential_process`,
and assume-role profiles are listed but cannot be copied into `[default]` —
`awsp` reports this rather than writing a broken profile. For those, use
`export AWS_PROFILE=...` or `aws sso login --profile ...`.

## License

MIT
