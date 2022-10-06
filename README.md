# assume-role

<img src="./assets/assume-role.png" align="right" alt="assume-role logo" />

Assume IAM roles through an **AWS Bastion** account with **MFA** via the command line.

**AWS Bastion** accounts store only IAM users providing a central, isolated account to manage their credentials and access. Trusting AWS accounts create IAM roles that the Bastion users can assume, to allow a single user access to multiple accounts resources. Under this setup, `assume-role` makes it easier to follow the standard security practices of MFA and short lived credentials.

## Installation

### Requirements

`assume-role` requires [`jq`](https://stedolan.github.io/jq/) and [`aws`](https://aws.amazon.com/cli/) CLI tools to be installed.

### Bash

1. Clone Repository
2. For easier updates create a symlink from your repository `assume-role` file to `/usr/local/bin/assume-role`
   
    **Example**

    ```bash
    ln -s <Git-Repository>/assume-role /usr/local/bin/assume-role
    ```

3. Add execution permissions
   
   ```bash
   chmod +x <Git-Repository>/assume-role
   ```




<!-- ### via Homebrew (macOS)

```bash
brew tap arvatoaws/assume-role
brew install assume-role
```

You can then upgrade at any time by running:

```bash
brew upgrade assume-role
``` -->

<!-- ### via Bash (Linux/macOS)

You can install/upgrade assume-role with this command:

```bash
curl https://raw.githubusercontent.com/arvatoaws-labs/assume-role/master/install-assume-role -O
cat install-assume-role # inspect the script for security
bash ./install-assume-role # install assume-role
```

It will ask for your sudo password if necessary. -->

## Getting Started

Make sure that credentials for your AWS bastion account are stored in `~/.aws/credentials`.

Out of the box you can call `assume-role` like:

```bash
eval $(assume-role account-id role mfa-token)
```

If your shell supports bash functions (e.g. zsh) then you can add `source $(which assume-role)` to your `rc` file (e.g. `~/.zshrc`), then you can call `assume-role` like:

<!-- ```bash
assume-role [account-id] [role] [mfa-token]
```

`assume-role` this method can be used with arguments or interactively like:

<img src="./assets/assume-role.gif" alt="assume-role usage" />

### Account Aliasing

You can define aliases to account ids in `~/.aws/accounts` which assume-role can use, e.g.

```json
{
  "default": "123456789012",
  "staging": "123456789012",
  "production": "123456789012"
}
```

With this file, to assume the `read` role in the `production` account:

```bash
assume-role production read
# OR
assume-role 123456789012 read
```

Also, by setting `$AWS_PROFILE_ASSUME_ROLE`, you can define a default profile for `assume-role` if you want to separate concerns between
default accounts for `assume-role` and vanilla `awscli` or simply to have better names than `default`:

```bash
$ export AWS_PROFILE_ASSUME_ROLE="bastion"
$ assume-role production read
```

Moreover, if you are in the need of [longer client-side assume-role sessions](https://aws.amazon.com/about-aws/whats-new/2018/03/longer-role-sessions/) and don't want to [enter your MFA authentication every hour (default)](https://github.com/coinbase/assume-role/issues/19) this one is for you:

```bash
$ export AWS_ROLE_SESSION_TIMEOUT=43200
```

However, be aware that for [chained roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_terms-and-concepts.html#iam-term-role-chaining) there's currently a forced **1 hour limit** from AWS. You'll get the following error if you exceed that specific limit:

> DurationSeconds exceeds the 1 hour session limit for roles assumed by role chaining.

## AWS Bastion Account Setup

Here is a simple example of how to set up a **Bastion** AWS account with an id `0987654321098` and a **Production** account with the id `123456789012`.

In the **Production** account create a role called `read`, with the trust relationship:

```json
{
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::0987654321098:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "true",
          "aws:MultiFactorAuthPresent": "true"
        },
        "NumericLessThan": {
          "aws:MultiFactorAuthAge": "54000"
        }
      }
    }
  ]
}
```

The conditions `aws:MultiFactorAuthPresent` and `aws:MultiFactorAuthAge` forces the use of temporary credentials secured with MFA.

In the **Bastion** account, create a group called `assume-read` with the policy:

```json
{
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [ "sts:AssumeRole" ],
      "Resource": [ "arn:aws:iam::123456789012:role/read" ],
      "Condition": {
        "Bool": {
          "aws:MultiFactorAuthPresent": "true",
          "aws:SecureTransport": "true"
        },
        "NumericLessThan": {
          "aws:MultiFactorAuthAge": "54000"
        }
      }
    }
  ]
}
```

Attach this group to **Bastion** users that should be able use `read`'s policies in the **Production** account.

You can assume the `read` role in **Production** by running:

```
assume-role 123456789012 read
```

Then entering a MFA token on request. -->

<!-- ## Prompt

If you are using `zsh` you can get a sweet prompt by adding to your `.zshrc` file:

```bash
source $(which assume-role)
# AWS ACCOUNT NAME
function aws_account_info {
  [ "$AWS_ACCOUNT_NAME" ] && [ "$AWS_ACCOUNT_ROLE" ] && echo "%F{blue}aws:(%f%F{red}$AWS_ACCOUNT_NAME:$AWS_ACCOUNT_ROLE%f%F{blue})%F$reset_color"
}

# )ofni_tnuocca_swa($ is $(aws_account_info) backwards
PROMPT=`echo $PROMPT | rev | sed 's/ / )ofni_tnuocca_swa($ /'| rev`
``` -->

## Auto autocompleter

If you want to have a autocompleter for the accounts from your aws-config add the following at the beginning of your `.zshrc` file:

### ZSH
Copy/link zsh function

```bash
ln -s <Git-Repository>/_assume_role ~/zsh_functions/_assume_role
```
### Bash
```bash
fpath=(~/zsh_functions $fpath)

autoload -U compinit
compinit
```

## ZSH Segments
If you are using oh-my-zsh, a nice way to integrate this into the powerline segments (the relevant one being the custom_assume_role, the other segmenst are merely an example) would be to do the following:
* Follow general assume-role instructions
* Setup oh-my-zsh normally
* Install Powerlevel10k Theme (https://github.com/romkatv/powerlevel10k#oh-my-zsh)
* Configure p10k (https://github.com/romkatv/powerlevel10k#get-started)
```bash
p10k configure
```
* Add the following to your .zshrc
```bash
source $(which assume-role)
export POWERLEVEL9K_LEFT_PROMPT_ELEMENTS=(root_indicator context dir dir_writable rbenv chruby nodeenv pyenv aws custom_assume_role vcs)
export POWERLEVEL9K_RIGHT_PROMPT_ELEMENTS=(status command_execution_time background_jobs detect_virt disk_usage load ram time)

export POWERLEVEL9K_CUSTOM_ASSUME_ROLE="echo \$AWS_ACCOUNT_NAME"
export POWERLEVEL9K_CUSTOM_ASSUME_ROLE_FOREGROUND="black"
export POWERLEVEL9K_CUSTOM_ASSUME_ROLE_BACKGROUND="yellow"

ZSH_THEME="powerlevel9k/powerlevel9k"
```


### Bash
For `bash` you could put the following in your `.bash_profile` file:

```bash
source $(which assume-role)

function aws_account_info {
  [ "$AWS_ACCOUNT_NAME" ] && [ "$AWS_ACCOUNT_ROLE" ] && echo -n "aws:($AWS_ACCOUNT_NAME:$AWS_ACCOUNT_ROLE) "
}

PROMPT_COMMAND='aws_account_info'
```


## YubiKey Integration

### Prerequisites
You have to install ykman for your distribution

### Installation

If you want to use your YubiKey as MFA, there is the feature to use the oath Feature of Yubikey:

You have to add your MFA Hash to oath:

```bash
ykman oath add -t NameOfYourChoice <YOUR_BASE_32_KEY>
```

After that you can add the following ENV Variable to your profile:

```bash
export YUBIKEY_MFA="NameOfYourChoice"
```

### Usage

Now, when assume-role needs a MFA it will ask you to Touch your YubiKey


<!-- ## Testing

assume-role is tested with [BATS](https://github.com/sstephenson/bats) (Bash Automated Testing System). To run the tests first you will need `bats`, `jq` and `shellcheck` installed. On macOS this can be accomplished with `brew`:

```bash
brew install bats
brew install jq
brew install shellcheck
```

Then run `bats test/assume-role.bats`; -->
