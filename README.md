GitAttributesDB
===============
This is a git hook to store/restore attributes (access/modification times, ownerships, permissions and - on Linux - ACLs, and xattrs) for paths stored in
a git repository, and for any extra paths configured for attribute store/restore.

This hook can be used in place of programs such as **etckeeper** to automatically (once set up) record and restore the attributes for paths in your `/etc`
directory.

I prefer this script over **etckeeper** as, once set up correctly, it is far simpler and completely automated - you do not need to run a command every time
you commit or pull changes to your `/etc` git repository.


Initial Set Up In New Repository
--------------------------------
Git hooks are usually stored in the `.git/hooks` directory inside the local repository, and are not pushed to the remote when you `git push` or kept under
version control.

As part of the initial set up, the hooks directory will be changed to be the `.githooks/` directory inside the repository root, and hooks inside that
directory put under version control - just like any paths in the repository.

The `gitattributesdb` git repository will be cloned under that directory as a git **submodule**, where the script can be called directly by the appropriate
hook files.

Firstly, create the `.githooks/` subdirectory in your current or new git repository:
```
mkdir .githooks/
```

And add the `gitattributesdb` repository as a submodule inside the `.githooks/` directory:
```
git submodule add https://github.com/tadgy/gitattributesdb.git .githooks/gitattributesdb
```

Once the `gitattributesdb` submodule is cloned, git hook scripts need to be added.

You may already have hooks stored in the `.git/hooks/` directory - these will need to be moved into the `.githooks/` directory.  
This command is only required if you already have hooks in your local copy of the repository:
```
mv .git/hooks/* .githooks/
```

`gitattributesdb` needs to be "hooked into" 3 git hook files: `post-checkout`, `post-merge` and `pre-commit`.
You may already have these files in the `.githooks/` directory, since they may have been moved from the `.git/hooks/` directory previously.

If you already have those files, you only need to add the syntax to run the `gitattributesdb` script to each of those hook files.  
Add the following in an appropriate place in those 3 files:
```
.githooks/gitattributesdb/gitattributesdb "${0##*/}" || exit $?
```

If those files do not already exist, you need to create and activate them:
```
touch .githooks/post-checkout .githooks/post-merge .githooks/pre-commit
chmod 755 .githooks/post-checkout .githooks/post-merge .githooks/pre-commit
```

Open each of the files `.githooks/post-checkout`, `.githooks/post-merge` and `.githooks/pre-commit` in your favourite text editor, adding the following to
each file:
```
#!/usr/bin/env bash

# Store/restore the attributes of files:
.githooks/gitattributesdb/gitattributesdb "${0##*/}" || exit $?
```
Save the changes to each file.

Configure git to use the new `.githooks/` directory rather than the default `.git/hooks` directory:
```
git config --local core.hooksPath .githooks
```

Add the submodule configuration and new hooks directory to the tracked paths of the repository - this puts all the hooks under version control:
```
git add .gitmodules .githooks/
```

Finally, unless you are setting up `gitattributesdb` in an existing repository (see below), commit the new paths into the repository:
```
git commit -m "Added .gitmodules file, and .githooks/ directory as the git hooks directory."
```

Initial set up of the repository to use `gitattributesdb` is complete.  
Whenever you commit changes to the repository, or pull new changes from a remote, the path attributes will be stored/restored.


Set Up In An Existing Repository
--------------------------------
Firstly, follow the instructions above in the "Initial Set Up In New Repository" section, but DO NOT perform the final `git commit` action.

Next, set the correct permissions, ownerships, ACLs and xattrs for the paths in your repository so that `gitattributesdb` can begin tracking them.

Add any paths you would like to be tracked by the `.gitattributesdb-extra` functionallity - see below for details on this functionallity.  Remember to
set the correct permissions, ownerships, ACLs and xattrs for these paths too.

Now that the paths in your repository are set to be tracked by `gitattributesdb`, you can go ahead an commit everything:
```
git commit -m "Added .gitmodules file, .githooks/ directory and check-in gitattributes database."
```

You can now work with your repository as normal and `gitattributesdb` will track the attributes from now on.


Set Up After A New Clone
------------------------
Git does **not** store the local repository configuration (stored in `.git/config`) on the remote when you push your changes.  This means that the
configuration to set the hooks directory is lost when the repository is cloned fresh.

It also does not automatically pull any embedded submodules into the repository when it is cloned.

In this situation, you need to have git pull the `gitattributesdb` submodule, and reconfigure the newly cloned repository to use the custom git hooks
directory:
```
git submodule update --init
git config --local core.hooksPath .githooks
```

This will clone the **exact** commit of `gitattributesdb` that was originally added to the repository - it does not track the branch itself, so changes at
the `HEAD` of the branch are not reflected in the submodule.  In order to get the latest changes, use the update procedure detailed below.

Once these commands have been run in the newly cloned repository (that has been initialised by the above procedure), everything is set for
`gitattributesdb` to maintain the attributes for paths.


Updating The Embedded `gitattributesdb` Submodule
-------------------------------------------------
From time to time it is a good idea to merge any changes from the remote branch into your local submodule of `gitattributesdb`.  
This allows you to pick up any fixes or updates to the tree.

To update the submodule **from the root of the git repository**, use:
```
(cd .githooks/gitattributesdb/ && git fetch && git merge origin/master)
```

The submodule will now have been updated to track the latest changes in the remote "master" branch.  The path (`.githooks/gitattributesdb/`) will need
to be checked into your repository with the next commit.


Tracking Extra Paths
--------------------
`gitattributesdb` has the ability to store/restore the attributes of extra paths on the filesystem that are **not** tracked in the git repository.

This is useful, for example, to track the attributes of `/etc/shadow`, without checking that file itself into git (and thus storing sensitive data in a
potentially publicly accessible git repository).

To achieve this, the path (relative to the root of the git repository) must be added to a special file, `.gitattributesdb-extra`, which should be placed
in the root of the repository.

To add paths to the "extra" files database, use:
```
{ printf "%s" <path> | base64 -w 0; printf "\\n"; } >>.gitattributesdb-extra
```
Where `<path>` is a path relative to the repository root.

Old paths (that no longer exist on the filesystem) stored in the `.gitattributesdb-extra` file are ignored when commiting.
