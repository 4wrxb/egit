# egit

POSIX-compatible shell script to use an existing directory in-place as the git worktree. Use cases include infrequent backup of directories where the .git folder is unwanted or embedded devices to direct the .git directory to a ramdisk.

Note that egit is *not* designed to manage sync with the remote repo. This can be dangerous since the .git directory is not necessairily persistent. It is important to push after comitting or making other changes, especially when using a ramdisk.

## Known Issues

egit is a work in progress and makes very odd use of git. Always be careful, but know the following don't work.

- Remote branches and tracking just disappear
- Merge and other branch-related commands in the repo tend to fail or have partial effects
- Hard reset commands don't always work

## Clone and Install

Clone this repository to the desired path, e.g.

```shell
egit_install="$HOME/git_work/egit"
git clone git@github.com:4wrxb/egit.git $egit_install/egit
```

If desired (i.e. working on a minute target already) use the [Bootstrap Clone]

Once you have your clone (however it is done) update path/aliases to avoid needing to type the full path.

```shell
# Path Usage (put this in an rc file to be permanent, may not work on all installs/shells due to escaping)
export $PATH="$egit_install:$PATH"

# Alias usage (put this in an rc file to be permannt)
alias egit="$egit_install/egit"

# Softlink usage (only needed once - assumes $HOME/bin/ exists and is in your path)
ln -s $egit_install/egit $HOME/bin/egit
```

### Bootstrap Clone

The egit repo is suitable for running egit directly. The project also contains a .egit.conf as an example. Although circular (and thus tricky) it's possible to use egit in order to manage its own install. For example, the following one-liner will clone egit to tmp and copy it into your $HOME/bin dir. From here either adjust your path, symlink egit, or set up an alias.

```shell
egit_install="$HOME/git_work/egit"
mkdir -p $egit_install && git clone https://github.com/4wrxb/egit.git /tmp/egit/egit_script_clone && cp -a /tmp/egit/egit_script_clone/* $egit_install/ && cp /tmp/egit/egit_script_clone/.egit.conf $egit_install

# Note: if you plan to develop egit after installing with this method you can update the origin so egit will continue to use this clone
git --git-dir /tmp/egit/egit_script_clone/.git/ remote set-url origin git@github.com:4wrxb/egit.git
```

## Usage

Once you have a repo configuration file containing the minimum [EGIT Config](#egit-config) simply replace git with egit and **only run commands from the repo root**. There is no support for upward path searching until a [Repo Configuration](#repo-configuration) file is found.

The egit script itself takes no arguments, all arguments are passed to git (along with additional arguments).

```shell
egit status

egit add example_file

egit commit -m "example commit"
```

Note that "gitk" support is not included as my current use case is with non-gui hosts. Any command that begins with "git" shold be OK (e.g. "egit citool") as long as it supports the arguments.

### Branches

Since the worktree outlives the git directory it can be inconvneinet to use checkout. In addition to the below command (which will switch branches without updating the worktree) egit supports specifying a branch for use in the current directory.

```shell
egit symbolic-ref HEAD refs/heads/otherbranch
egit reset
```

To specify the branch for use *on future downloads* use the `EGIT_BRANCH` variable in the [Repo Configuration]. If the egit work area for this directory already exists the above command must be used to switch branches.

### Refrence Mode

By default egit's work area contains a directory for every model with its own objects etc. even if the origin is the same. To support refrences when using the same origin egit must explicitly run in refrence mode.

Refrence mode first clones a bare "refrence" repository of the origin. This can be a given name (by setting EGIT_REFNAME) or it can be a hash of the origin's path (when EGIT_REFNAME = 'AUTO'). Note that AUTO will be affected by variations in access methods (i.e. https vs ssh or different path names).

Refrence mode is WIP and not all commands may behave as expected.

## Configuration

egit configuration  is comprised of 3 parts. These parts are parsed in order so that the last (most specific) config overrides earlier settings.

1) System Configuration file egit.conf (an example is included in the repo)
2) User Configuration file
3) Repo Configuration file .egit.conf (an example is included in the repo for egit itself)

File locations and details are covered below

Configuration files are simple POSIX-compatible shell scripts (and can use test conditionals etc.. Generally, there are two types of configuration they contain

- Local variables meant for the egit script itself. See [EGIT Config](#egit-config)
- Environment variables meant for git itself. Useful since the repo configuration file is not persistent. See [Git documentation](https://git-scm.com/book/en/v2/Git-Internals-Environment-Variables)

### EGIT Config

The egit config are local shell variables.

| Config        | Required? | Default Value     | Description                                                                                  |
| ------------- | --------- | ----------------- | -------------------------------------------------------------------------------------------- |
| EGIT_DEBUG    | No        | 0                 | Set to 1 to enable debug output                                                              |
| EGIT_WORK     | No - Warn | /tmp/egit         | Will issue a warning of not explicit in a config.                                            |
| EGIT_ORIGIN   | Yes       | N/A               | Set this to a git remote, same as `git clone <remote>`.                                      |
| EGIT_BRANCH   | No        |                   | Set the branch name for *new* nowloads of the egit work area of this directory               |
| EGIT_SUMCMD   | No        | See below         | The command to generate a checksum from stdin. Default will find an appropriate *sum binary. |
| EGIT_DIRKEY   | No        | See below         | Sets the name or relative path (inside $EGIT_WORK) of the bare repo.                         |
| EGIT_REFNAME  | No        |                   | Use Refrence mode (with given name). Set to "AUTO" to use hash of `EGIT_ORIGIN`.             |
| USR_EGIT_CONF | No        | `$HOME/egit.conf` | Sets path of the user's egit config. Only applies in system config file.                     |
| DIR_EGIT_CONF | No        | `./.egit.conf`    | Sets the path of the repo's egit config. USE CAUTION: this should be a relative path.        |

#### EGIT_WORK

The EGIT_WORK location is set to `/tmp/egit` in order to take advantage of the (typically) default ramdisk on embedded devices. This can be modified if there is a desire for the .git to be presistent.

#### EGIT_DIRKEY

The EGIT_DIRKEY is appended to EGIT_WORK to determine the location of the bare repo (the same as a .git folder, but not named .git). By default a checksum of the real path of worktree/PWD (`readlink -f $PWD`) is generated using the configured/detected [Checksum Tool](#checksum-tool) but this can be specified manually.

For auto-calculated dirkey's the real worktree path is also saved to egit_worktree.path for readability (since all directory names are just hashes) as well as a sanity check for checksum colissions. In the case of configured dirkeys **this check is bypassed** in order to support advanced use-cases.

The primary use-case of overriding the EGIT_DIRKEY would be if the same bare repo can be shared between different folders on the same host. Note that in this case you are NOT just sharing objects (egit does not support git clone --reference) you are sharing the entire git dir. This includes branches, references, etc.. Use caution when setting the EGIT_DIRKEY to ensure the EGIT_ORIGIN matches, otherwise EGIT will remove the directory to replace it. **There is no check whether an existing directory is synced with its origin**.

### System Configuration File

egit pulls a system "egit.conf" from one of two locations, depending on the egit script's location.

- If egit is running from /usr/bin or /bin then /etc/egit.conf is used
- If egit is running from another directory it is assumed to be a dev install and $SCRIPT_DIR/egit.conf is used

### User Configuration File

The user configuration file is loaded from $HOME/egit.conf **unless the variable "USR_EGIT_CONF" is overridden by the system configuration file**. This file can be useful in cases where you wish to avoid a global git config or use egit with a different author than your global git config.

For example, to set the author/comitter name and email when you don't have a .gitconfig

**$HOME/egit.conf**
```shell
# Set author so git stops complaining
export GIT_AUTHOR_NAME="Rex"
export GIT_AUTHOR_EMAIL="4wrxb@example.com"
export GIT_COMMITTER_NAME=$GIT_AUTHOR_NAME
export GIT_COMMITTER_EMAIL=$GIT_AUTHOR_EMAIL
```

### Repo Configuration

The repo (directory) configuration file is the only required configuration file. It must contain the `EGIT_ORIGIN` setting for egit to work. See [EGIT Config](#egit-config) for details.

By default the repo configuration file is called ".egit.conf" and must be in the directory where the command is run (which will also be the work-tree root of the git command). **This can be overridden in the system/user config files by setting DIR_EGIT_CONF**.

For example, a basic config would contain only the first line. Additional lines give an example of using shell scripting to override the commit author if the hostname contains router.

**./.egit.conf**
```shell
EGIT_ORIGIN="git@github.com:4wrxb/mybackup.git"
EGIT_BRANCH="erx"

# Only use egit reference if we aren't working in root (our build setup has branches for each router)
if [ "$(readlink -f "$PWD")" != "/" ]; then
  EGIT_REFNAME="AUTO"
fi

# Override author of commits on routers
if $(grep -q "router" /proc/sys/kernel/hostname); then
  export GIT_AUTHOR_NAME="$(cat /proc/sys/kernel/hostname) backup"
  export GIT_AUTHOR_EMAIL="router@example.com"
  export GIT_COMMITTER_NAME=$GIT_AUTHOR_NAME
  export GIT_COMMITTER_EMAIL=$GIT_AUTHOR_EMAIL
fi
```

egit's own repo has a more complete example configuration file which works with the repo itself.

## Checksum Tool

egit uses a checksum for its directory keys to identify repos stored in the temp area (see [EGIT_DIRKEY](#egit_dirkey) for details). For maximum portability as many tools as possible are supported, even if they are known to cause issues with collisions etc.. The search iterats most common *sum tools. If any are missing please suggest through issues etc..

Note that the sum tool can be specified manually (i.e. if not in your path variable) using EGIT_SUMCMD. It is recommended to set this at a user/system level since egit will warn if the checksum tool does not match previous runs (i.e. differs per egit repo).

### Workarounds for Single Repo Mode

If egit is unable to find a appropriate cksum tool a hurestic can be applied through system config. Although the safest method is to specify the dirkey for each egit repo this may not be feasible.

EGIT_SUMCMD will apply before cksum search, so it's not recomended in any corss-platfrom repo configs where it may override valid sum tools. The setting EGIT_SUMCMD will be eval'd with input to be "hashed" passed through stdin. For example, a simple huristic would be to replace each directory seperator with '_' (although this can cause collisions).

```shell
EGIT_SUMCMD="sed 's#/#_#g'"
```

It is also possible to use directories to cut down on colissions, but this can cause path-length issues

```shell
EGIT_SUMCMD="sed 's#/#_/#g'"
```

Any number of tools can apply using eval-able statements. This will come down to the applicable use-case.

Note that worktree checks **will** apply to custom SUMCMDs (but not to EGIT_DIRKEY).

## Debugging

Since the egit script takes no arguments for convenience sake there is no commad-based ability to enable debugging. Instead the EGIT_DEBUG variable must be set to 1 in a config file. If there are issues loading config files the edit the egit file itself.
