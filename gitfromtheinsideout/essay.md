# Git from the inside out

This essay shows you how Git works.  It focuses on the graph that underpins Git and how the properties of this graph dictate the behaviours of Git.  This focus on fundamentals lets you build your mental model on the truth, rather than on hypotheses constructed from evidence gathered while experimenting with the API.  This truer model gives you a better understanding of what Git has done, what it is doing and what it will do.

The text is structured as a series of Git commands run on a single project.  At intervals, there are explorations of the Git graph.  These explain the properties of the graph that are pertinant to the commnd being run, and how those properties control the behaviour of the command.

This essay assumes you understand Git well enough to use it to version control your projects.

## Create the project

```bash
$ mkdir alpha
$ cd alpha
```

The user creates the project, a directory called `alpha`.

# Initialize a repository

```bash
$ git init
  Initialized empty Git repository
```

`git init` turns the current directory into a Git repository.  To do this, it creates a `.git` directory and writes some files to it.  These files define everything about the configuration and history of a project.  They are just ordinary files.  No magic in them.  The user can read and change them with a text editor or shell.  Which is to say: the user can read and edit the history of their project as easily as their project files.

## Add some files

```bash
$ mkdir data/
$ printf '1' > data/number.txt
```

The user adds a file, `data/number.txt`, to the project.  It contains the text `1`.  The `.git` directory and its contents are Git's.  All the other files are collectively known as the working copy.  They're the user's.

```
$ git add data/number.txt
```

The user runs `git add` on the `data/number.txt` file.  This has two effects.

First, it creates a new blob file in the directory at `alpha/.git/objects/`.

This file contains the compressed contents of the `data/number.txt` file.  The compression is nothing special - it's just like zip and bzr and the other types of compression used for large text files.

The name of this file is derived by hashing the file's contents.  "Hashing" a piece of text means running a program on it that turns it into a smaller* piece of text that uniquely* identifies the original.  For example, Git hashes `1` to `43dd47ea691c90a5fa7827892c70241913351963`.  This hash is a short, unique identifier for the current content of `number.txt`.  The hash is actually split across a directory inside `alpha/.git/objects/` - `43` - and a file inside that directory - `dd47ea691c90a5fa7827892c70241913351963`*.

Notice how just adding a file to Git saves its content to the objects directory.  If the user were to delete the `data/number.txt` file from the hard drive, its contents would still be safe inside Git.

Second, `git add` adds the file to the index.  The index is a list that contains every file that Git has been told to keep track of.  It is just a file that lives at `alpha/.git/index` [SAY HOW TO list INDEX]. Each line of the file maps a tracked file to a hash of its contents at the moment it was added.

```bash
$ printf 'q' > data/letter.txt
```

The user makes a file called `data/letter.txt` that contains `q`.  The project looks like this:

```text
alpha
└── data
    └── letter.txt
    └── number.txt
```

```bash
$ git add data/
```

The user runs `git add`.  This creates a blob file that contains the content of `data/letter.txt`.  And it adds another index entry that maps the `data/letter.txt` file to a hash of its contents.

```bash
$ printf 'a' > data/letter.txt
$ git add data/letter.txt
```

When the user originally created `data/letter.txt`, they meant to type `a`, not `q`.  They make the correction and add the file to the index again.  This creates a new blob with the new content.  It updates the index entry for `data/letter.txt` so it maps to the hash of the latest content.

## Make a commit

```bash
$ git commit -m 'a1'
  [master (root-commit) c388d51] a1
```

Committing has three steps.  It creates a tree graph.  It creates a commit object.  It updates the ref of the current branch.

### Create a tree graph

Git records the current state of the project by creating a tree graph from the index*[advs tree graph].  This tree graph records the location and content of every file and directory in the project.

The graph is composed of two types of object: blobs and trees.  These objects are just more text files stored in the `alpha/.git/objects/` directory.

Blobs are the objects that were stored by `git add`.  They represent the content of files.

Trees are stored during commits.  They represent a directory in the project.  There is one line for each item in the directory.  Each line records the four things required to reproduce a file or directory in the project.  The item's permissions.  The type of object (blob or tree) that represents the item.  The hash of the object.  The name of the item.

Below is the tree object that records the state of the `data` directory.  It has entries for the `number.txt` and `letter.txt` files.  Notice that the entries use hashes to point to the blob objects that represent their content.

```
100664 blob hhh letter.txt
100664 blob hhh number.txt
```

Below is the tree object for `alpha`, the root directory of the project.  It has a pointer to the `data` tree.

```
040000 tree hhh data
```

Notice how directories are incidental to Git.  After the user ran `git add data/`, the index had an entry for each file.  The `data/` directory is not listed separately.

```
data/letter.txt hhh
data/number.txt hhh
```

This means that empty directories will not appear in any tree graph.  This is what people mean when they say, "Git only tracks files."

[nice picture of alpha tree pointing at number blob and doc tree and doc tree pointing at readme blob]

### Create a commit object

After creating the tree graph, `git commit` creates a commit object.  This is just another text file in the `.git/objects/` directory.  It looks like this:

```
tree hhh
author Mary Rose Cook <mary@maryrosecook.com> 1424798436 -0500
committer Mary Rose Cook <mary@maryrosecook.com> 1424798436 -0500

a1
```

The first line is a pointer to the tree graph that represents the content of the index at the moment the commit was made.  This pointer is actually to the root of the tree graph: the tree object that represents the `alpha` directory.

The last line is the commit message.

The Git graph now looks like this:

[drawing of repo graph with commit object]

### Update the commit the current branch points to

Finally, the commit command points the current branch at the new commit object.

What is the current branch? To find out, Git goes to the `HEAD` file at `alpha/.git/HEAD` and finds:

```
ref: refs/heads/master
```

It finds a path to a file called `master`.  This indicates that `master` is the current branch.  This file does not exist, because this is the first commit to the repository.  Git creates the file and writes to it the hash of the commit object:

```
hhh
```

Here are `HEAD` and `master` on the git graph:

[drawing of repo graph with HEAD and master]

`HEAD` still points at `master`.  But `master` now exists and points to the new commit object.

## Make a commit that is not the first commit

```bash
$ printf '2' > data/number.txt
$ git add data/number.txt
```

The user sets the content of `data/number.txt` to `2` and adds the file.  This adds a blob containing `2`.  It updates the index with the hash of the new version of `data/number.txt`:

```
letter.txt hhh
number.txt hhh[new]
```

```bash
$ git commit -m 'a2'
  [master ae78f19] a2
```

The user commits.  The steps for the commit are the same as before.

First, a new tree graph is created to represent the content of the index.

A new tree object is created to represent the `data` directory.  This must be created because the hash of the contents of the `data/number.txt` file has changed.

```
100664 blob hhh letter.txt
100664 blob hhh number.txt
```

A new tree object is created to represent the `alpha` directory.  This must be created because the hash of the `data` tree object has changed.

```
040000 tree hhh data
```

Second, a new commit object is created that points at the new tree object.

```
tree hhh
parent hhh
author Mary Rose Cook <mary@maryrosecook.com> 1424813101 -0500
committer Mary Rose Cook <mary@maryrosecook.com> 1424813101 -0500

a2
```

The second line of the commit object points at the commit's parent: the previous commit.  To find the parent commit, Git went to `HEAD`, followed it to `master` and found the old commit hash.

Third, just like for the previous commit, the contents of the `master` branch file is set to the hash of the new commit.

***

[graph with `a2` commit with the information sharing]

One.  Content is stored as a tree of objects.  When a tree graph is created for a new commit, Git only creates trees and blobs that do not already exist.  In this way, Git stores only diffs of content, rather than storing the same content again and again.  Because there are few changes from commit to commit, Git can store large commit histories in a small amount of space.  Look at the tree graph for the `a2` commit.  It reuses the `a` blob that was created before the `a1` commit.

Two.  Each commit has a parent.  This creates the history of a repository.  To see the history preceding a commit, just move from parent to parent, all the way back to the first commit.

Three.  The nodes in the graph are all text files.  This means they can be easily retrieved, edited, converted and explored with programs other than Git.

Five.  Refs are entry points to one part of the commit history or another.  The user uses concrete refs like `fix-for-bug-376` to organise their work into lineages that are meaningful to their project.  Git uses symbolic refs like `HEAD`, `MERGE_HEAD` and `FETCH_HEAD` to support commands that can manipulate the commit history: committing, merging, fetching.

Six.  The nodes in the `objects/` directory are immutable.  This means that Git allows edits, not deletions.  Every piece of content ever added and every commit ever made is in there somewhere\*[does garbage collect chuck out stuff?].

Seven.  Refs are immutable.  Though the commit that `master` points at now might be the best version of a project, soon enough, it will be superceded be a newer and better commit.

Eight.  Recent history is easier to recall, but more changeable.  Or: Git has a fading memory that must be jogged with increasingly vicious prods.

The working copy is the easiest point in history to recall because it is in the root of the repository.  Recalling it doesn't even require a Git command.  It is also the least permanent.  The user can make a dozen versions of a file, but, unless they are added, Git won't record any of them.

The commit that `HEAD` points is very easy to recall.  It is at the tip of the branch that is checked out.  To see its content, the user can just stash[*explain this] and then examine the working copy.  At the same time, `HEAD` is the most frequently changing ref.

The commit that a concrete ref points at is easy to recall.  The user can simply check out that branch.  The tips of branches change less frequently that `HEAD`, but still frequently enough for them to be ephemeral.

It is possible to recall a commit that is not pointed at by any ref.  The further back the user goes, the harder it will be for them to sort through the content and reassemble the meaning of a commit in their human brain.  But, the further back they go, the less likely it is that someone will have changed history since they last looked*.

***

## Check out a commit

```bash
$ git checkout hhh
  You are in 'detached HEAD' state...
```

The user checks out the `a2` commit usings its hash.  Checking out has four steps.

First, Git gets the `a2` commit and gets the tree graph it points at.

Second, it writes the file entries in the tree graph to the files of the working copy.  This results in no changes.  Because `HEAD` was already pointing (via `master`) at the `a2` commit, the working copy already has the same content as the tree graph.

Third, it writes the file entries in the tree graph to the index.  This, too, results in no changes.  The index already has the same content as the `a2` commit.

Fourth, the contents of `HEAD` to the hash of the `a2` commit:

```
hhh
```

[graph showing head pointing directly at commit]

```bash
$ printf '3' > data/number.txt
$ git add data/number.txt
$ git commit -m 'a3'
  [master 45c0be2] a3
```

The user sets the content of `data/number.txt` to `3` and commits the change.  To get the parent of the `a3` commit, Git follows the detached `HEAD` directly to the previous `a2` commit, rather than going via a branch.  This means that the new commit is not on a branch.

[graph of new commit not on a branch]

## Create a branch

```bash
$ git branch deputy
```

The user creates a new branch called `deputy`.  This just creates a new file at `alpha/.git/refs/heads/deputy` that contains `a3`, the hash that `HEAD` is pointing at.

***

Nine.  A ref, like `deputy` and `HEAD`, is a file that contains a hash that identifies a commit in the Git graph.  It is computationally cheap to create and modify these files.  THis is why it is often said that Git branches are lightweight.

***

The creation of the `deputy` branch puts the new `a3` commit safely on a branch.

## Check out a branch

```bash
$ git checkout master
  Switched to branch 'master'
```

The user checks out the `master` branch.

First, Git gets the `a2` commit that `master` points at and gets the tree graph the commit points at.

Second, it writes the file entries in the tree graph to the files of the working copy.  This sets the content of `data/number.txt` to `2`.

Third, it writes the file entries in the tree graph to the index.  In this case, the index becomes:

```
100664 blob hhh number.txt [hhh should show content for `2`]
```

Fourth, it points `HEAD` at `master`:

```
ref: refs/heads/master
```

## Check out a branch that is incompatible with the working copy

```bash
$ printf 'asdf' > data/number.txt
$ git checkout deputy
  Your changes to these files would be overwritten by checkout:
	data/number.txt
  Please, commit your changes or stash them before you switch branches.
```

The user accidentally sets the content of `data/number.txt` to `asdf`.  They try and check out `deputy`.  Git prevents the check out.

`HEAD` points at `master` that points at a commit where `data/number.txt` reads `2`.  `deputy` points at the commit where `data/number.txt` reads `3`.  For simplicity, Git only allows check outs that do not require a merge to resolve.  Files in the working copy must either be new, or have the same content as the current commit, or have the same content as the commit being checked out.

```bash
$ printf '2' > data/number.txt
$ git checkout deputy
  Switched to branch 'deputy'
```

The user notices that they accidentally edited `data/number.txt` and sets the contents back to `2`.  They check out `deputy` successfully.

## Merge an ancestor

```bash
$ git merge master
  Already up-to-date.
```

The user merges `master` into `deputy`.  Merging two branches means merging two commits.  There is the commit that `master` points at: the giver.  And there is the commit that `deputy` points at: the receiver.  For this merge, Git does nothing, reporting it is `Already up-to-date.`.

***

[graph that shows that merged commit was ancestor of receiver]

Nine.  The series of commits added to the history are interpreted as a series of changes made to the content of the repository.  This means that, if the giver commit is an ancestor of the receiver commit, Git will do nothing.  Those changes have already been incorporated.

***

## Merge a descendent

```bash
$ git checkout master
  Switched to branch 'master'
$ git merge deputy
  Fast-forward
```

The user checks out `master`.  They merge `deputy` into `master`.  Git discovers that the receiver commit is an ancestor of the giver commit.  It does a fast-forward merge.

Git gets the receiver commit and gets the tree graph that it points at.  It writes the file entries in the tree graph to the working copy and the index.  It points `master` at the giver commit.  This makes `a3` the current commit.

***

[graph that shows that merged commit was descendent of current]

Ten.  The history is a series of changes.  This means that, if the giver is a descendent of the receiver, history is not changed.  There is already a sequence of commits that describe the change to make: the sequence of commits between the receiver and the giver.  But, though the Git history doesn't change, the Git graph does change.  The concrete ref that `HEAD` points at is moved to point at the commit being merged.

***

## Merge a commit from a different lineage

```bash
$ printf '4' > data/number.txt
$ git add data/number.txt
$ git commit -m 'a4'
  [master 45c0be2] a4
```

The user sets the content of `number.txt` to `4` and commits the change to `master`.

```bash
$ git checkout deputy
  Switched to branch 'deputy'
$ printf 'b' > data/letter.txt
$ git add data/letter.txt
$ git commit -m 'b4'
  [master 45c0be2] b4
```

The user checks out `deputy`.  They set the content of `data/letter.txt` to `b` and commit the change to `deputy`.

***

[graph that shows that merged commit and current commit have different lineages]

Eleven.  Commits can share parents.  This means that new lineages can be created in the commit history.

Twelve.  The commit history can contain multiple lineages.  This means that, if the user merges two commits in different lineages, those lineages must be joined.  To join two lineages, Git creates a merge commit.

***

```bash
$ git merge master
  Merge made by the 'recursive' strategy.
```

The user merges `master` into `deputy`.

Git discovers that the giver and receiver are in different lineages.  It makes a merge commit.  This process has six steps.

First, Git writes the hash of the giver commit to a file at `alpha/.git/MERGE_HEAD`.  The presence of this file tells Git it is in the middle of merging.

Second, Git creates a diff that contains the changes required to go from the receiver commit to the giver commit.  This diff is a list of file paths that point to a change: add, remove, modify or conflict.  Git finds the base commit: the most recent common ancestor of the receiver and giver commits.

***

[graph showing the merge base commit]

Thirteen.  Git stores the parents of commits.  This means it can find the point at which two lineages diverged.

***

Git generates the indices for the receiver, giver and base commits.  It gets the list of all the files that appear in at least one of the indices.  For each one, it compares the index entries to see what change was made to the file.  It writes a corresponding entry to the diff.

In this case, the diff has two entries.

The first is for `data/letter.txt`.  The content of the file is `a` in the base, `b` in receiver and `a` in the giver.  The content is different in the base and receiver.  But it is the same in the base and giver.  This means that Git can see that the content was only modified by the giver, not the receiver.  Which means that the diff entry for `data/letter.txt` is a modification, not a conflict.

The second entry in the diff is for `data/number.txt`.  In this case, the content is the same in the base and receiver, and different in the giver.  This means that the diff entry for `data/letter.txt` is also a modification, not a conflict.

***

[graph showing the base, receiver and giver and their content]

Fourteen.  Git can find the base commit of a merge.  If a file has changed from that base version in just the receiver or giver, Git can automatically resolve the merge of that file.  This means lees work for the user.

***

Third, the changes indicated by the entries in the diff are applied to the index.  This means that the entry for `data/letter.txt` is pointed at the `b` blob and the entry for `data/number.txt` is pointed at the `4` blob.

Fourth, the changes indicated by the entries in the diff are applied to the working copy.  This means that the content of `data/letter.txt` is set to `b` and the content of `data/number.txt` is set to `4`.

Fifth, the updated index is committed:

```
tree 140706a7ce6843548bfb6b3411c0a11cb135bc36
parent hhh
parent hhh
author Mary Rose Cook <mary@maryrosecook.com> 1425596551 -0500
committer Mary Rose Cook <mary@maryrosecook.com> 1425596551 -0500

Merge branch 'master' into deputy
```

Notice that the commit has two parents.

Sixth, the current branch, `master`, is set to point at the new commit.

## Merge two commits in different lineages that both modify the same file

```bash
$ printf '5.3' > data/number.txt
$ git add data/number.txt
$ git commit -m 'a5.3'
  [master 45c0be2] a5.3
```

The user sets the content of `data/number.txt` to `5.3` and commits the change to `master`.

```bash
$ git checkout `deputy`
  Switched to branch 'deputy'
$ printf '5.7' > data/number.txt
$ git add data/number.txt
$ git commit -m 'a5.7'
  [master 45c0be2] a5.7
```

The user checks out `deputy`.  They set the content of `data/number.txt` to `5.7` and commit the change to `deputy`.

```bash
$ git merge master
  CONFLICT in data/number.txt
  Automatic merge failed; fix conflicts and then commit the result.
```

The user merges `master` into `deputy`.  There is a conflict and the merge is paused.  The process for a conflicted merge follows the same first four steps as the process for an unconflicted merge: set `alpha/.git/MERGE_HEAD`, create a diff, update the index and update the working copy.  Because of the conflict, steps two, three and four have different outcomes.  Because of the conflict, the fifth commit step and sixth ref update step are never taken.  Let's go through the steps again and see what happened.

First, Git writes the hash of the giver commit to a file at `alpha/.git/MERGE_HEAD`.  This is the same as before.

Second, Git creates a diff that contains the changes required to go from the receiver commit to the giver commit.  In this case, the diff contains only one entry: `data/number.txt`.  Because the content for `data/number.txt` is different in the receiver, giver and base, the entry is a conflict.

Third, the changes indicated by the entries in the diff are applied to the index.  Entries in the index are uniquely identified by a combination of their file path and stage.  The entry for an unconflicted file has a stage of `0`.  Before this merge, the index looked like this, where `0` is the stage:

```
0 data/letter.txt hhh
0 data/number.txt hhh
```

After the diff is written to the index, it looks like this:

```
0 data/letter.txt hhh
1 data/number.txt hhh [hash for base]
2 data/number.txt hhh [hash for receiver]
3 data/number.txt hhh [hash for giver]
```

The entry for `data/letter.txt` at stage `0` is still there.  The entry for `data/number.txt` at stage `0` is gone.  There are three new entries in its place.  The entry for stage `1` has the hash of the `data/number.txt` content from the base commit.  The entry for stage `2` has the hash of the `data/number.txt` content from the receiver commit.  The entry for stage `3` has the hash of the `data/number.txt` content from the giver commit.  The presence of these three entries tells Git that `data/number.txt` is in conflict.

Fourth, the changes indicated by the entries in the diff are applied to the working copy.  For a conflict, Git writes both versions to the working copy file.  The content of `data/number.txt` is set to:

```
<<<<<<< HEAD
5.7
=======
5.3
>>>>>>> master
```

The merge pauses here.

```bash
$ printf '6' > data/number.txt
$ git add data/number.txt
```

The user edits `data/number.txt` to resolve the conflict and sets its content to `6`.  They add the file to the index.  Adding a conflicted file tells Git that the conflict is resolved.  Git removes the `data/number.txt` entries for stages `1`, `2` and `3` from the index and adds an entry for stage `0`.  The index now contains:

```
0 data/letter.txt hhh
0 data/number.txt hhh
```

```bash
$ git commit -m `b6`
  [master 45c0be2] b6
```

The user commits.  Git sees `alpha/.git/MERGE_HEAD` in the repo, which tells it that a merge is in progress.  It checks the index and finds there are no conflicts.  It creates a new commit to record the content of the resolved merge.  It deletes the file at `alpha/.git/MERGE_HEAD`.  This completes the merge.

[graph showing the new merge commit]

## Remove a file

```bash
$ git rm data/letter.txt
  rm 'data/letter.txt'
```

The user tells Git to remove the `data/letter.txt` file.

First, `data/letter.txt` is deleted from the working copy.

Second, the entry for `data/letter.txt` is deleted from the index.

```bash
$ git commit -m '6'
  [master 45c0be2] 6
```

The user commits.  As part of the commit, as always, Git builds a tree graph that represents the content of the index.  Because `data/letter.txt` is not in the index, it is not included in the tree graph.

## Copy a repository

```bash
$ git checkout master
  Switched to branch 'master'
$ cd ..
$ cp -r alpha bravo
```

The users checks out `master`.  They make a copy of the entire `alpha/` repository to the `bravo/` directory.

## Link a repository ta another repository

```bash
$ cd alpha
$ git remote add bravo ../bravo
```

The user moves back into the `alpha` repository.  They set up `bravo` as a remote repository on `alpha`.  This adds some lines to the file at `alpha/.git/config`:

```
[remote "bravo"]
	url = ../bravo/
```

These lines specify that there is a remote repository called `bravo` in the directory at `../bravo`.

## Fetch a branch from a remote

```bash
$ cd ../bravo
$ printf '7' > data/number.txt
$ git add data/number.txt
$ git commit -m '7'
  [master 45c0be2] 7
```

The user goes into the `bravo` repository.  They set the content of `data/number.txt` to `7` and commit the change to `master` on `bravo`.

```bash
$ cd ../alpha
$ git fetch bravo master
  Unpacking objects: 100%
  From ../bravo
    * branch master -> FETCH_HEAD
```

The user goes into the `alpha` repository.  They fetch `master` from `bravo` into `alpha`.  This takes four steps.

First, Git gets the hash of the commit that master is pointing to on `bravo`.  This is the hash of the `7` commit.

Second, a list is made of all the objects that the `7` commit depends on: the commit itself, the objects in its tree graph, the ancestor commits of the `7` commit and the objects in their tree graphs.  It copies all these objects to `alpha/.git/objects/`.

Third, the contents of the concrete ref file at `alpha/.git/refs/remotes/bravo/master` is set to the hash of the `7` commit.

***

[graph showing bravo/master pointing at `7` commit (somehow show the fact that the commit is not part of the local history]

Fifteen.  Objects can be copied.  This means that parts of the content and history of a project can be copied from one repository to another.

Sixteen.  A repository can store remote branch refs like `alpha/.git/refs/remotes/bravo/master`.  This means a repository can have a record of the state of a branch on a remote repository.  Though correct the last time it was fetched, if the remote repository changes, it might go out of date.

***

Fourth, the contents of `alpha/.git/FETCH_HEAD` is set to:

```
hhh	branch 'master' of ../bravo
```

This indicates that the most recent fetch command fetched the `7` commit of `master` on `bravo`.

## Merge FETCH_HEAD

```bash
$ git merge FETCH_HEAD
  Updating ac79079..dd8807b
  Fast-forward
```

The user merges `FETCH_HEAD`.  `FETCH_HEAD` is just another ref.  It resolves to the `7` commit, the giver.  `HEAD` points at the `6` commit, the receiver.  Git does a fast-forward merge and points `master` at the `7` commit.

## Pull a branch from a remote

```bash
$ git pull bravo master
  Already up-to-date.
```

The user pulls `master` from `bravo` into `alpha`.  Pulling is shorthand for fetching and merging `FETCH_HEAD`.  Git does these two commands and reports that `master` is `Already up-to-date`.

## Clone a repository

```bash
$ cd ../
$ git clone alpha charlie
  Cloning into 'charlie'
```

The user moves into the directory above.  They clone `alpha` to `charlie`.  Cloning has similar results to the `cp` the user did to produce the `bravo` repository.

Git creates a new directory called `charlie`.  After that, it inits `charlie` as a Git repo, adds `alpha` as a remote called `origin`, fetches `origin` and merges `FETCH_HEAD`.

## Push a branch to a checked out branch on a remote

```bash
$ cd ../alpha
$ printf '8' > data/number.txt
$ git add data/number.txt
$ git commit -m '8'
  [master 45c0be2] 8
```

The user goes back into the `alpha` repository.  They set the content of `data/number.txt` to `8` and commit the change to `master` on `alpha`.

```bash
$ git remote add charlie ../charlie
```

They set up `charlie` as a remote repository on `alpha`.

```bash
$ git push origin master
  Writing objects: 100%
  remote error: refusing to update checked out branch: refs/heads/master
                because it will make the index and work tree inconsistent
```

They push `master` to `charlie`.

All the objects required for the `8` commit on the `master` branch are copied to `charlie`.

At this point, the push process stops.  Git, as ever, tells the user what went wrong.  It refuses to push to a branch that is checked out on the remote.  This makes sense.  A push will update the current commit and index of the remote.  If someone is editing the working copy on the remote, this will be confusing.

At this point, the user could make a new branch and push that branch to `charlie`.  But, really, they want `charlie` be a repository they can push to however they want.  They want `charlie` to be a central repository that they can push to and pull from, but that no one commits to directly.  They want something like a GitHub remote.  They want a bare repository.

## Clone a bare repository

```bash
$ cd ..
$ rm -rf charlie
$ git clone alpha charlie --bare
  Cloning into bare repository 'charlie'
```

The user deletes the `charlie` repository.  They clone `charlie` as a bare repository.  This is an ordinary clone with two differences.  The `config` file indicates the repository is bare.  And the contents of the `.git` directory are in the top of the repository:

```
charlie
├── HEAD
├── config
├── objects
└── refs
```

## Push a branch to a bare repository

```bash
$ cd ../alpha
$ printf '9' > data/number.txt
$ git add data/number.txt
$ git commit -m '9'
  [master 45c0be2] 9
```

The user goes back into the `alpha` repository.  They set the content of `data/number.txt` to `9` and commit the change to `master` on `alpha`.

```bash
$ git push charlie master
  Writing objects: 100%
  To ../charlie
    dd8807b..ec904de  master -> master
```

They push `master` to `charlie`.  Pushing has three steps.

First, all the objects required for the `8` commit on the `master` branch are copied from `alpha/.git/objects/` to `charlie/.git/objects/`.

Second, `refs/heads/master` is updated on `charlie` to point at the `9` commit.

Third, `.git/refs/heads/remotes/charlie/master` is updated to point at the `9` commit.  `alpha` has to update its own record of the state of `charlie`.

## Summary

Git is built on a graph.  Almost every Git command manipulates this graph.  To understand Git deeply, focus on the properties of this graph, not workflows or commands.

To learn more about Git, investigate the `.git` directory.  It's not scary. Look inside.  Change the contents of files and see what happens.  Create a commit by hand.  Try and see how badly you can mess up a repo.  Then repair it.

--------------


todo
- go through all commands, check they work, add their output exactly to bash sections
- show current dir!
- find a way to put file names on same line as top border of code samples

git graph:
- show graphic that allows switching between history view and content view of object graph?
- later: can do a git graph that includes the wc graph coming off head and the index coming off head?
- fade out lines to show omitted irrelevant nodes of git graph?
- show changes somehow? (New/changed things in green?)
