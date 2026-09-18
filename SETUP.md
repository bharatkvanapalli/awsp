# Setting up AWS accounts on a new machine

Two ways to reach an AWS account from the command line. This page covers both, plus how
to switch between them with `awsp`. Nothing here needs copying from an old machine.

| | Path A: Identity Center | Path B: access keys |
|---|---|---|
| Use when | The account has a sign-in portal | You were handed a key, or the account has no portal |
| You store | No secret at all | A key and secret, sometimes a session token |
| Lifetime | Expires, renewed by signing in | Permanent, or minutes to hours when temporary |
| Daily cost | One `aws sso login` | Nothing, until a key needs rotating |
| Setup below | [Path A](#path-a-iam-identity-center) | [Path B](#path-b-access-keys-with-or-without-a-session-token) |

Mixing both is normal: environment accounts on path A, a client's or a personal account
on path B.

## Install once

```sh
brew install awscli jq
git clone https://github.com/bharatkvanapalli/awsp.git
install -m 755 awsp/awsp ~/bin/awsp
grep -q 'HOME/bin' ~/.zshrc || echo 'export PATH="$HOME/bin:$PATH"' >> ~/.zshrc
```

Two files hold every setting:

| File | Holds |
|---|---|
| `~/.aws/config` | Profiles, regions, Identity Center sessions, `awsp` aliases |
| `~/.aws/credentials` | Access keys and session tokens, nothing else |

Never copy `~/.aws/credentials` between machines. Issue a new key on the new machine and
delete the old one, so a lost laptop costs one key rather than all of them.

## Path A: IAM Identity Center

Use this whenever the account sits behind a sign-in portal. One wizard per account:

```sh
aws configure sso --profile dev
# SSO session name: company          any name; other profiles reuse it
# SSO start URL: https://d-xxxxxxxxxx.awsapps.com/start
# SSO region: us-east-1
# SSO registration scopes: sso:account:access
# the browser opens, then pick the account and the role, then name the profile
```

The same thing without the wizard, when the ids are already known:

```sh
aws configure set sso_session company --profile dev
aws configure set sso_account_id 123456789012 --profile dev
aws configure set sso_role_name AdministratorAccess --profile dev
aws configure set region us-east-1 --profile dev
```

Sign in once per day:

```sh
aws sso login --sso-session company
```

Every profile on that session is then live, so one sign-in covers many accounts. The
credentials each command receives are short-lived and refresh on their own.

**One portal user per environment?** Give each its own session block pointing at the same
start URL. Tokens cache per session name, so several users can be signed in at once:

```ini
[sso-session company-dev]
sso_start_url = https://d-xxxxxxxxxx.awsapps.com/start
sso_region = us-east-1
sso_registration_scopes = sso:account:access
```

Each session needs its own `aws sso login`. The browser reuses the portal cookie of
whoever signed in last, so sign in to a second user from a private window.

## Path B: access keys, with or without a session token

```sh
aws configure --profile clientB
# AWS Access Key ID:      paste
# AWS Secret Access Key:  paste
# Default region name:    us-east-1
# Default output format:  json
```

Repeat per account, changing only the profile name. The name is just a label you choose;
the key itself decides which account you land in, so confirm with
`aws sts get-caller-identity --profile clientB`.

**Temporary credentials** from an assumed role or a portal's copy-paste block carry a
third value:

```sh
aws configure set aws_session_token "PASTE_TOKEN" --profile clientB
```

These expire between 15 minutes and 36 hours, after which commands fail with "The security
token included in the request is invalid" and you paste a fresh set. The expiry is not
stored anywhere, so nothing warns you first.

When a profile moves from temporary to permanent credentials, delete the stale token line
or every call fails, blaming the key rather than the token:

```sh
python3 - <<'PY'
import configparser, os
path = os.path.expanduser('~/.aws/credentials')
cfg = configparser.RawConfigParser(); cfg.read(path)
cfg.remove_option('clientB', 'aws_session_token')
cfg.write(open(path, 'w'))
PY
```

If you paste temporary credentials often, replace the paste with something that refreshes
itself: a profile with `role_arn` plus `source_profile`, or path A.

## Switching between them

```sh
awsp                          # every profile with its alias, kind and account
awsp dev                      # exact name, alias, or a unique substring
awsp --login                  # sign in to the current profile's SSO session
aws sts get-caller-identity   # the proof of where you actually are
awsp --clear                  # no default; every command needs --profile
```

Give any profile a short name once:

```sh
aws configure set awsp_alias prod --profile some-long-profile-name
```

Two things silently beat the default, and `awsp` warns about both: an `AWS_PROFILE`
exported in that shell, and a `[default]` section holding credentials in
`~/.aws/credentials`.

## Rotating a permanent key

An IAM user may hold two access keys at once, which is what makes rotation possible
without locking yourself out. Create, switch, verify, then delete:

```sh
aws iam create-access-key --user-name someone --profile clientB
aws configure --profile clientB                       # paste the new pair
aws sts get-caller-identity --profile clientB         # proves the new key works
aws iam delete-access-key --user-name someone --access-key-id OLD --profile clientB
```

For Identity Center profiles there is nothing to rotate. That is the point of path A.
