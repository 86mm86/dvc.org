# install

Install DVC hooks into the Git repository to automate certain common actions.

## Synopsis

```usage
usage: dvc install [-h] [-q | -v]
```

## Description

DVC provides an intelligent data repository on top of a regular SCM repository
like Git to store code and configuration files. With `dvc install`, the two are
more tightly integrated in order to cause certain convenient actions to happen
automatically.

Namely:

**Checkout**: For any given SCM branch or tag, Git checks out the
[DVC-files](/doc/user-guide/dvc-file-format) corresponding to that version. The
DVC-files in turn refer to data files in the DVC cache by checksum. When
switching from one SCM branch or tag to another, the SCM retrieves the
corresponding DVC-files. By default that leaves the workspace in a state where
the DVC-files refer to data files other than what is currently in the workspace.
The user at this point should run `dvc checkout` so that the data files will
match the current DVC-files.

The installed Git hook automates running `dvc checkout`.

**Commit**: When committing a change to the Git repository, that change possibly
requires reproducing the corresponding [pipeline](/doc/get-started/pipeline)
(with `dvc repro`) to regenerate the workspace results. Or there might be files
not yet in the cache, which is a reminder to run `dvc commit`.

The installed Git hook automates reminding the user to run either `dvc repro` or
`dvc commit`.

**Push**: While publishing changes to the Git remote repository with `git push`,
it easy to forget that `dvc push` command usually needs to be run to save
corresponding changes in data files and directories that are under DVC control
to the DVC remote storage.

The installed Git hook automates executing `dvc push`.

## Installed Git hooks

- Git `pre-commit` hook executes `dvc status` before `git commit` to inform the
  user about the workspace status.
- Git `post-checkout` hook executes `dvc checkout` after `git checkout` to
  automatically synchronize the data files with the new workspace state.
- Git `pre-push` hook executes `dvc push` before `git push` to upload files and
  directories under DVC control to remote.

For more information about git hooks, refer to the
[git-scm documentation](https://git-scm.com/docs/githooks).

## Disable Git hooks

When you run `dvc install`, it creates three files under the `.git/hooks`
directory:

```
.git/hooks
├── post-checkout
├── pre-commit
└── pre-push
```

To disable them, you need to **remove** or **edit** those files (i.e.
`rm .git/hooks/post-checkout`, `vim .git/hooks/pre-commit`).

_If a hook already exists, DVC will try to append the corresponding content, so
if you want to disable the DVC-specific behavior, you'll need to open the script
and edit it._

## Options

- `-h`, `--help` - prints the usage/help message, and exit.

- `-q`, `--quiet` - do not write anything to standard output. Exit with 0 if no
  problems arise, otherwise 1.

- `-v`, `--verbose` - displays detailed tracing information.

## Examples

To explore `dvc install` let's consider a simple pipeline with several stages:
the example workspace used in the [Getting Started](/doc/get-started) tutorial.

<details>

### Click and expand to setup the project

This step is optional, and you can run it only if you want to run this examples
in your environment. First, you need to download the project:

```dvc
$ git clone https://github.com/iterative/example-get-started
```

Second, let's install the requirements. But before we do that, we **strongly**
recommend creating a virtual environment with
[virtualenv](https://virtualenv.pypa.io/en/stable/) or a similar tool:

```dvc
$ cd example-get-started
$ virtualenv -p python3 .env
$ source .env/bin/activate
```

Now, we can install requirements for the project:

```dvc
$ pip install -r requirements.txt
```

Then download the precomputed data using:

```dvc
$ dvc pull --all-branches --all-tags
```

This data will be retrieved from a preconfigured remote cache.

</details>

## Example: Checkout both DVC and Git

Let's start our exploration with the impact of `dvc install` on the
`dvc checkout` command. Remember that switching from one SCM tag or branch to
another changes the set of DVC-files in the workspace, which then also changes
the data files that should be in the workspace.

With the Getting Started example workspace described above, let's first list the
available tags:

```dvc
$ git tag

0-empty
1-initialize
2-remote
3-add-file
4-sources
5-preparation
6-featurization
7-train
8-evaluation
9-bigrams
baseline-experiment
bigrams-experiment
```

These tags are used to mark points in the development of this workspace, and to
document specific experiments conducted in the workspace. To take a look at one
we check-out the workspace using the SCM (in this case Git):

```dvc
$ git checkout 6-featurization
Note: checking out '6-featurization'.

You are in 'detached HEAD' state.  ...

$ dvc status

featurize.dvc:
    changed outs:
        modified:           data/features

$ dvc checkout

[##############################] 100% Checkout finished!

$ dvc status

Pipeline is up to date. Nothing to reproduce.
```

After running `git checkout` we are also shown a message saying _You are in
'detached HEAD' state_, and the Git documentation explains what that means.
Bottom line is returning the workspace to a normal state requires the command
`git checkout master`.

We also see that `dvc status` tells us about differences between the workspace
and the data files currently in the workspace. Git changed the DVC-files in the
workspace, which changed references to data files. What `dvc status` did is
inform us the data files in the workspace no longer matched the checksums in the
DVC-files. Running `dvc checkout` then checks out the corresponding data files,
and now `dvc status` tells us the data files match the DVC-files.

```dvc
$ git checkout master
Previous HEAD position was d13ba9a add featurization stage
Switched to branch 'master'
Your branch is up to date with 'origin/master'.

$ dvc checkout
[##############################] 100% Checkout finished!
```

We've seen the default behavior with there being no Git hooks installed. We want
to see how the behavior changes after installing the Git hooks. We must first
reset the workspace to he at the head commit before installing the hooks.

```dvc
$ dvc install

$ cat .git/hooks/pre-commit
#!/bin/sh
exec dvc status

$ cat .git/hooks/post-checkout
#!/bin/sh
exec dvc checkout
```

The two Git hooks have been installed, and the one of interest for this exercise
is the `post-checkout` script which runs after `git checkout`.

We can now repeat the command run earlier, to see the difference.

```dvc
$ git checkout 6-featurization
Note: checking out '6-featurization'.

You are in 'detached HEAD' state. ...

HEAD is now at d13ba9a add featurization stage
[##############################] 100% Checkout finished!

$ dvc status

Pipeline is up to date. Nothing to reproduce.
```

Look carefully at this output and it is clear that the `dvc checkout` command
has indeed been run. As a result the workspace is up-to-date with the data files
matching what is referenced by the DVC-files.

## Example: Showing DVC status on Git commit

The other hook installed by `dvc install` runs before `git commit` operation. To
see see what that does, start with the same workspace, making sure it is not in
the detached HEAD state from the previous example.

If we simply edit one of the code files:

```dvc
$ vi src/featurization.py

$ git commit -a -m "modified featurization"

featurize.dvc:
    changed deps:
        modified:           src/featurization.py
[master 1116ddc] modified featurization
 1 file changed, 1 insertion(+), 1 deletion(-)
```

We see that `dvc status` output has appeared in the `git commit` interaction.
This new behavior corresponds to the Git hook which was installed, and it
helpfully informs us the workspace is out of sync. We should therefore run the
`dvc repro` command.

```dvc
$ dvc repro evaluate.dvc

... much output
To track the changes with git run:

    git add featurize.dvc train.dvc evaluate.dvc

$ git status -s
 M auc.metric
 M evaluate.dvc
 M featurize.dvc
 M src/featurization.py
 M train.dvc

$ git commit -a -m "updated data after modified featurization"

Pipeline is up to date. Nothing to reproduce.

[master 78d0c44] modified featurization
 5 files changed, 12 insertions(+), 12 deletions(-)
```

After reproducing this pipeline up to the "evaluate" stage, the data files are
in sync with the code/config files, but we must now commit the changes to the
Git repository. Looking closely we see that `dvc status` is run again, informing
us that the data files are synchronized with the `Pipeline is up to date.`
message.
