# encpass.sh

encpass.sh provides a lightweight solution for using encrypted passwords in shell scripts. It allows a user to encrypt a password (or any other secret) at runtime and then use it, decrypted, within a script. This prevents shoulder surfing secrets and avoids storing the secret in plain text, which could inadvertently be sent to or discovered by an individual at a later date.

The default OpenSSL implementation generates an AES 256 bit symmetric key for each script (or user-defined bucket) that stores secrets. This key will then be used to encrypt all secrets for that script or bucket.

Subsequent calls to retrieve a secret will not prompt for a secret to be entered as the file with the encrypted value already exists.

Note: By default, encpass.sh sets up a directory (.encpass) under the user's home directory where keys and secrets will be stored.  This directory can be overridden by setting the environment variable ENCPASS_HOME_DIR to a directory of your choice.

~/.encpass (or the directory specified by ENCPASS_HOME_DIR) will contain the following subdirectories:

* keys (Holds the private key for each script or user-defined bucket)
* secrets (Holds the secrets stored for each script or user-defined bucket)
* exports (Holds any export files created by the export command)

## Requirements

encpass.sh requires the following software to be installed:

* POSIX compliant shell environment (sh, bash, ksh, zsh)
* OpenSSL or Extension replacement (See the "Extensions" section)

Note: Even if you use fish or other types of modern shells, encpass.sh should still be usable as those shells typically support running POSIX compliant scripts.  You just won't be able to include encpass.sh in any fish specific scripts or other non-POSIX compliant scripts.

## Installation

Download the encpass.sh script and place it in a directory on your path.

Example: curl the script to /usr/local/bin
```
$ curl https://raw.githubusercontent.com/ahnick/encpass.sh/master/encpass.sh -o /usr/local/bin/encpass.sh
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  3085  100  3085    0     0   5184      0 --:--:-- --:--:-- --:--:--  5193
```

## Usage

Source encpass.sh in your script and call the get_secret function.

```
#!/bin/sh
. encpass.sh

password=$(get_secret)
echo $password
```

See [example.sh](examples/example.sh) for examples of calling the get_secret function using a named secret or a named secret for a specific bucket.


## Command Line Management

encpass.sh also provides a command line interface to perform various management functions, such as:
* Add secrets/buckets
* Remove secrets/buckets
* List secrets/buckets
* Show secrets/buckets
* Lock/Unlock all keys for buckets
* Import/Export secrets/buckets

For example, the following command will list the secrets stored for example.sh:
```
$ encpass.sh ls example.sh
password
```

Use the help command to list all the available commands along with detailed descriptions:
```
$ encpass.sh ?
```

## Lite Version

The command line management functions while useful, do significantly increase the size of encpass.sh.  In cases where scripts only need to get or set secrets, then a smaller version of encpass.sh would be more convenient for inclusion with existing scripts.  A smaller (lite) version of encpass.sh can be generated by calling the lite command and then redirecting the output to a file.
```
$ encpass.sh lite > encpass-lite.sh
```

This will create a truncated copy of encpass.sh with all of the command line management functions completely removed.


## Configuration

By default, encpass.sh will create a hidden directory in the user's home directory to store all it's data (keys and secrets) the first time it is run.  This means there is no configuration and setup required to begin using encpass.sh.  Nevertheless, there are certain optional configuration changes that can be made to further improve the user experience.


### Set a different encpass home directory using ENCPASS_HOME_DIR

A location other than ~/.encpass can be used for the encpass home directory by setting the ENCPASS_HOME_DIR environment variable.

For example, you could store the ENCPASS_HOME_DIR directory on an encrypted filesystem or networked mount point such as KBFS (Keybase Filesystem).  To do this, place the following export command (substituting your own username) in your shell configuration file (e.g. ~/.bashrc or ~/.bash_profile for BASH), so that every time you start a shell, encpass.sh will point to this directory.
```
export ENCPASS_HOME_DIR="/keybase/private/<USER>/.encpass"
```

To quickly display the current value of ENCPASS_HOME_DIR in your shell use the **dir** command
```
$ encpass.sh dir
ENCPASS_HOME_DIR=/keybase/private/<USER>/.encpass
```

### Maintain a list of encpass home directories using ENCPASS_DIR_LIST

A colon delimited list of directories can be specified in the environment variable ENCPASS_DIR_LIST.  This provides a useful way of tracking multiple encpass home directories.
```
export ENCPASS_DIR_LIST="/home/<USER>/.encpass:/keybase/private/<USER>/.encpass"
```

When the ENCPASS_DIR_LIST environment variable is set then you can use the **dir ls** command to quickly list all your encpass home directories.
```
$ encpass.sh dir ls
/home/<USER>/.encpass
/keybase/private/<USER>/.encpass
```

This makes it easier to find the available encpass home directories and then point encpass to a different one by setting the ENCPASS_HOME_DIR environment variable.  Changing encpass home directories is even easier if you add the "ehd" function to your shell from the next section.

### Command completion, aliasing, and encpass home directory function

If you regularly use encpass.sh from the command line to maintain your secrets, at some point you may want to reduce the amount of typing required.  Here are several recommendations you can add to your shell configuration file to make things faster.

Note, all these examples are using Bash syntax, but other shells should have similar equivalents.  If you have commands for other shells, feel free to add them to an rc file for that shell under the [rc](rc/) directory and submit a pull request.  We'll be happy to review and incorporate it into the repo.


**Add command line completion to your shell. (Currently only bash command completion exists)**
Copy the [completion](completion/bash/completion) to your ENCPASS_HOME_DIR and source the file
```
source $ENCPASS_HOME_DIR/completion
```

Now when you type your commands you type the first few characters and then hit \<TAB\> for the command to complete.  Bonus, this works for subcommands, buckets and secrets as well! :)


**Alias encpass.sh to ep** 
```
alias ep="encpass.sh"
```

Command completion still works for the "ep" alias as well.  You can use whatever alias you would like, but if you want the command completion to work with a different alias you will need to update the completion script.


**Add an encpass home directory function**
```
ehd() { export ENCPASS_HOME_DIR="$1"; [ -f "$1/completion" ] && . "$1/completion"; }
```

When you call "ehd" with the path of an encpass home directory it will set ENCPASS_HOME_DIR and then enable command line completion automatically if a completion file exists.


**Putting it altogether the final [bashrc](rc/bashrc) config looks as follows..**
```
# encpass.sh
alias ep="encpass.sh"
export ENCPASS_HOME_DIR="$HOME/.encpass"
export ENCPASS_DIR_LIST="$HOME/.encpass" # colon delimited list of dirs
ehd() { export ENCPASS_HOME_DIR="$1"; [ -f "$1/completion" ] && . "$1/completion"; }
ehd $ENCPASS_HOME_DIR
```

This sets up our alias, exports our environment variables, adds our change directory function and calls the function to enable command completion.


## Backing up secrets/keys

If you'd like to make a backup copy of your secrets/keys, you can create a script to call the export command to export all your secrets/keys to a gzip compressed tar archive. (.tgz)  This can then easily be copied to Dropbox or your backup storage of choice.

You can optionally encrypt the exported file with a password.  This password can even be stored within encpass itself to make automation simple.  (It is recommended that you keep a copy of this password separately in a secure location to ensure you can always open the archive later)

Please see the [backup-encpass.sh](examples/backup-encpass.sh) for such an example.


## Extensions

encpass.sh can be extended to provide additional capabilities or change existing functions.  One of the primary uses of the extension facility is to override the default encryption backend (OpenSSL) and implement a different backend.

The following extensions currently exist for encpass.sh:
* [Keybase](extensions/keybase/KEYBASE.md) (Encrypts/Decrypts secrets using Keybase user/team keys and stores them in Keybase encrypted remote repositories) 

Only one extension can be enabled for one ENCPASS_HOME_DIR to ensure there are no unexpected side effects with multiple extensions enabled at once.

If you'd like to make a new extension, the easiest way is to copy the Keybase extension and rename it using the format encpass-*extension*.sh.  Then modify the script by adding your logic to existing sections or add new sections.  You can override most functions of the main encpass.sh script in an extension script.

Once your extension is complete create a subfolder for your extension under the [extensions](extensions/) directory and place your script in it.  If you would like your extension added to the official encpass.sh repository just open a pull request and we'll be happy to review it. :)


## Testing with Docker
All testing can be carried out using the Makefile bundled with the repo.  This Makefile will build and run a docker image to carry out the testing; therefore, you must have docker installed on your local machine in order to execute the tests.

To verify that any changes made do not contain any errors, please run shellcheck using make.
```
make check
```
If there are any errors output please correct those errors or suppress the warnings if appropriate. 

encpass.sh also strives to be usable in all POSIX compliant shell environments (i.e. SH, BASH, ZSH, KSH).  To verify changes to the script have not broken compliance, you can run the bundled unit tests with the repo using make. 
```
make test
```


## Important Security Information

Although encpass.sh encrypts every secret on disk within the user's home directory, once the password is decrypted within a script, the script author must take care not to inadvertently expose the password. For example, if you invoke another process from within a script that is using the decrypted password AND you pass the decrypted password to that process, then it would be visible to ps.

Imagine a script like the following...
```
#!/bin/sh
. encpass.sh
password=$(get_secret)
watch whatever.sh --pass=$password &
ps -A
```

Upon executing you should see the password in the ps output...
```
97349 ??         9:56.30 watch whatever.sh --pass=P@$$w0rd
```
