# awsp

Switch the **global** default AWS profile with one short command, so every terminal,
script and IDE hits the account you picked. Works with IAM Identity Center (SSO),
assume-role and access-key profiles, and never copies a credential anywhere.

```console
$ awsp

default -> acme-dev
  123456789012  arn:aws:sts::123456789012:assumed-role/AWSReservedSSO_AdministratorAccess_1a2b/you

profiles
  * acme-dev                 sso      123456789012
    acme-qa                  sso      210987654321
    acme-prod                sso      345678901234
    legacy                   keys

usage: awsp <profile> | awsp --login | awsp --clear

$ awsp qa
default -> acme-qa  sso 210987654321
  210987654321  arn:aws:sts::210987654321:assumed-role/AWSReservedSSO_AdministratorAccess_3c4d/you
```

`awsp qa` matched `acme-qa` on a substring, so environment names are usually enough.

## Short names

When a profile's real name is long or does not contain the word you think in, give it an
alias once:

```sh
aws configure set awsp_alias prod --profile company-platform-production-sso
aws configure set awsp_alias personal --profile my-own-account
```

`awsp prod` then picks that profile, whatever it is called. Aliases show in the first
column of `awsp`. Resolution order is exact profile name, then alias, then a unique
substring; if a substring matches several profiles, a single SSO one wins.

## One user per environment

If your Identity Center has a separate user per environment, give each environment its
own `[sso-session]` block pointing at the same start URL. Tokens are cached per session
name, so all of them can be signed in at once and `awsp` switches between them without
a new sign-in:

```ini
[sso-session company-dev]
sso_start_url = https://d-xxxxxxxxxx.awsapps.com/start
sso_region = us-east-1
sso_registration_scopes = sso:account:access

[profile company-dev-sso]
sso_session = company-dev
sso_account_id = 123456789012
sso_role_name = AdministratorAccess
awsp_alias = dev
```

Each session needs `awsp --login` once. The browser may reuse the portal cookie of the
user you signed in as last, so use a private window when signing in as a different one.

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

## Commands

| Command | Does |
|---|---|
| `awsp` | Show the current default, its live identity, and every profile with its kind and account id |
| `awsp <profile>` | Make that profile the default; a unique substring is enough |
| `awsp -i`, `awsp --login` | Run `aws sso login` for the current default's session |
| `awsp -l`, `awsp --list` | Profile names only, one per line |
| `awsp -c`, `awsp --clear` | Remove `[default]` so every command needs `--profile` again |

## Install

```sh
git clone https://github.com/bharatkvanapalli/awsp.git
install -m 755 awsp/awsp ~/bin/awsp     # any directory on your PATH
```

Requires the AWS CLI v2 (2.9 or newer, for `aws configure export-credentials`) and
python3, which macOS already has.

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
- Repos that pin a profile per environment (for example Terraform wrappers that pass
  `--profile` explicitly) ignore the default, by design.

## License

MIT, see [LICENSE](LICENSE).
