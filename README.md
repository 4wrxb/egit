# egit

POSIX-compatible shell script to use an existing directory in-place as the git worktree. Use cases include infrequent backup of directories where the .git folder is unwanted or embedded devices to direct the .git directory to a ramdisk.

Note that egit is *not* designed to manage sync with the remote repo. This can be dangerous since the .git directory is not necessairily persistent. It is important to push after comitting or making other changes, especially when using a ramdisk.

## Usage

Once you have a repo configuration file containing the minimum [EGIT Config] simply replace git with egit and **only run commands from the repo root**. There is no support for upward path searching until a [Repo Configuration] file is found.

The egit script itself takes no arguments, all arguments are passed to git (along with additional arguments).

```shell
egit status

egit add example_file

egit commit -m "example commit"
```

Note that git-gui support (including "gitk" and "git citool") is not included as my current use case is with non-gui hosts.

## Configuration

egit configuration  is comprised of 3 parts. These parts are parsed in order so that the last (most specific) config overrides earlier settings.

1) System Configuration file
2) User Configuration file
3) Repo Configuration file (MUST contain the EGIT_ORIGIN setting)

Configuration files are simple POSIX-compatible shell scripts (and can use test conditionals etc.. Generally, there are two types of configuration they contain

- Local variables meant for the egit script itself. See [EGIT Config]
- Environment variables meant for git itself. Useful since the repo configuration file is not persistent. See [Git documentation](https://git-scm.com/book/en/v2/Git-Internals-Environment-Variables)

### EGIT Config

The egit config are local shell variables.

| Config      | Required? | Default Value     | Description                                               |
| ----------- | --------- | ----------------- | --------------------------------------------------------- |
| EGIT_DEBUG  | No        | 0                 | Set to 1 to enable debug output                           |
| EGIT_WORK   | No - Warn | /tmp/egit         | Will issue a warning of not explicit in a config.         |
| EGIT_ORIGIN | Yes       | N/A               | Set this to a git remote, same as `git clone <remote>`    |
| EGIT_DIRKEY | No        | `$PWD -> s#/#_#g` | Determines the name (inside $EGIT_WORK) of the bare repo  |

#### EGIT_WORK

The EGIT_WORK location is set to `/tmp/egit` in order to take advantage of the (typically) default ramdisk on embedded devices. This can be modified if there is a desire for the .git to be presistent.

#### EGIT_DIRKEY

The EGIT_DIRKEY is appended to EGIT_WORK to determine the location of the bare repo (the same as a .git folder, but not named .git). Typically the real path of PWD (`readlink -f $PWD`) is used with all `/` replaced with `_` (this includes the leading `/` so root is simply _).

The primary use-case of overriding the EGIT_DIRKEY would be if the same bare repo can be shared between different folders on the same host.

Use caution when setting the EGIT_DIRKEY to ensure the EGIT_ORIGIN matches, otherwise EGIT will remove the directory to replace it. **Currently there is no check whether an existing directory is synced with its origin**.

### System Configuration File

egit pulls a system "egit.conf" from one of two locations, depending on the egit script's location.

- If egit is running from /usr/bin or /bin then /etc/egit.conf is used
- If egit is running from another directory it is assumed to be a dev install and $SCRIPT_DIR/egit.conf is used

### User Configuration File

The user configuration file is loaded from $HOME/egit.conf unless the variable "USR_EGIT_CONF" is overridden by the system configuration file. This file can be useful in cases where you wish to avoid a global git config or use egit with a different author than your global git config.

For example, to set the author/comitter name and email

**$HOME/egit.conf**
```shell
# Set author so git stops complaining
export GIT_AUTHOR_NAME="Rex"
export GIT_AUTHOR_EMAIL="4wrxb@example.com"
export GIT_COMMITTER_NAME=$GIT_AUTHOR_NAME
export GIT_COMMITTER_EMAIL=$GIT_AUTHOR_EMAIL
```

### Repo Configuration File

The repo (directory) configuration file is the only required configuration file. It must contain the `EGIT_ORIGIN` setting for egit to work. See [EGIT Config] for details.

By default the repo configuration file is called ".egit.conf" and must be in the directory where the command is run (which will also be the work-tree root of the git command). This can be overridden in the system/user config files by setting DIR_EGIT_CONF.

For example, a basic config would contain only the first line. Additional lines give an example of using shell scripting to override the commit author if the hostname contains router.

**./.egit.conf**
```shell
EGIT_ORIGIN="git@github.com:4wrxb/mybackup.git"

# Override author of commits on routers
if $(grep -q "router" /proc/sys/kernel/hostname); then
  export GIT_AUTHOR_NAME="$(cat /proc/sys/kernel/hostname) backup"
  export GIT_AUTHOR_EMAIL="router@example.com"
  export GIT_COMMITTER_NAME=$GIT_AUTHOR_NAME
  export GIT_COMMITTER_EMAIL=$GIT_AUTHOR_EMAIL
fi
```

## Debugging

Since the egit script takes no arguments for convenience sake there is no commad-based ability to enable debugging. Instead the EGIT_DEBUG variable must be set to 1 in a config file. If there are issues loading config files the edit the egit file itself.
