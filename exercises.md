#Grok Git Exercises

##Exercise 1 - Storing Content
Let’s look at the files created in a git directory.  Creating from scratch so you can see I have nothing up my sleeve…
```
mkdir trinug
cd trinug
git init
```

Now we have an empty repository.  All git files are stored in .git hidden folder.  Let’s take a look...
```
ls .git
```
The objects folders is where all of our content is stored.  Remember it’s a simple key-value store.  Let’s add some content:

```
echo "Angular Fundamentals” > presentations.txt
```

Now we have a file in our working directory.  Remember, git doesn’t track and could care less about your working directory.
Still nothing in our objects folder:
find .git/objects -type f

Let’s add presentations.txt with a plumbing command:
git hash-object -w presentations.txt
665a7c3
-w means write
The hexadecimal that comes back is the hash.
We only need the first 7 characters to reference this hash with git.

This file is now in the objects directory.
ls .git/objects

Notice the new folder named 66?  That’s the first two hex characters of our sha. Let’s look in that folder:

find .git/objects -type f
Nice.  The file name is the rest of the characters of the sha.  So now we can see how git stores our data in files and folders under the objects folder.

Let’s prove our sha will always be the same for that content by reading the text from standard in instead of the file:
echo 'Angular Fundamentals' | git hash-object --stdin

Look ma, same hash!

Now let’s look this value up with another plumbing command:
git cat-file -p 665a7c3

What does our repository look like now?
<*******************   Slide **************************>

What if we change the content of this file?
echo “JavaScript Basics” >> presentations.txt

But we still only have one file in our objects folder:
find .git/objects -type f

Let’s add the second version of this document to our objects folder:
git hash-object -w presentations.txt
9dd4a1f

Notice the new hash value?  Now let’s see the content of our objects folder:
find .git/objects -type f

Now we have both versions of the file in there!
<*******************   Slide **************************>

So now it’s an object in the repository, but it’s not referenced.  How do we fix that?
Let’s update the index (our staging area):
git update-index --add --cacheinfo 100644 665a7c307051c1fcf350be1217ca7f0344639ce8 presentations.txt

The index is a binary file that keeps track of everything in the staging area:
ls .git
What’s in the index file?
git ls-files --stage

Notice the index file here that was not present before.
<*******************   Slide  **************************>
Why does git have a working directory and an index?

- Allows you to make partial commits.
- Gives you a place to work outside of the content system.

<*******************   Slide **************************>
Up to this point we have no tree to track our files or folders.
use the write-tree command to write the staging area out to a tree object:
git write-tree
find .git/objects -type f
git cat-file -p d131452

We have a tree file which points to one object file (our first version of presentations.txt).
<*******************   Slide **************************>

Now let’s create a commit:
echo 'my first commit' | git commit-tree d131452

and review the commit:
git cat-file -p <SHA>

Commits enable storage of history.

That looks familiar!  A commit is just a file that contains some meta-data and points to our top level tree object.  It has some other meta data like author (from the config), the committer and the associated timestamps.  In this case the author and committer of this commit are the same.
<*******************   Slide **************************>

A commit is just another object in the data store:
find .git/objects -type f

So let’s look at git status to see what’s going on:
git status

Wait a minute, our file is staged, but it doesn’t seem to be committed?  We created a commit but we didn’t point our current branch (master) to it.  Let’s do that now...
git update-ref refs/heads/master <SHA>

let’s check it now
git status
git log
<*******************   Slide **************************>
Awesome, we just did some of our normal workflow with the plumbing commands!
Add our objects to the repo, stage them in the index, write out a tree object, create a commit pointing to that tree object (and any parent), and then move our head branch to point to it.

Now let’s add some more objects to make our working directory more interesting, but this time we’ll use the porcelain commands we use everyday:
mkdir codecamp
cd codecamp
echo ‘ASP.NET Core’ > sessions.txt
cd ..
git add -A
git commit -m "adding a list of sessions"

Those commands are a nice shortcut to what we did above, but it’s really helpful to understand how git is storing our data!
<*******************   Slide **************************>
Now, let’s look at our latest commit:

git cat-file -p <sha>
The top level tree:
git cat-file -p 31af022
sessions tree:
git cat-file -p bbbd3a83ff

Note: Commits could (and mostly do) point to existing objects from earlier commits.

Let’s switch gears and look at how git handles branches and change history.
Graphs git complicated fast.  We’ll just show commits.

<*******************   Slide **************************>
Branches

- Higher level DAG concerned with commits
- Just a pointer to a commit
- Master is a the default name
- Head points to the current active branch

ls .git/head
cat .git/head
cat .git/refs/heads/master

Fast creation of branches!  Notice the sha here?  This is just a pointer to the last commit.
Why is our branch named master?  There is nothing special here, it’s the out-of-the-box branch with git, but you could name it anything if you wanted.

Let’s create a new branch:

git branch add-sessions
<*******************   Slide **************************>
check out the new branch and notice it points to the same place as master:

more .git/refs/heads/add-sessions

let’s checkout this branch and make a change to the presentations file:

git checkout add-sessions
echo "Docker" >> presentations.txt
git add -A
git commit -m “added Docker"

<*******************   Slide **************************>
Now we can see that the add-more-session branch has diverged from master:
more .git/refs/heads/master
more .git/refs/heads/add-sessions

Remembering the commits where these pointed, let’s merge add-more-sessions back into master
git checkout master
git merge add-sessions
<*******************   Slide **************************>
This created a fast-forward merge.  This is an optimization git uses when there are no diverging changes between the branches.  In this case git can just move (fast forward) the master branch reference forward to point to the same commit as add-more-sessions.  No merge commit is necessary.  Let’s take a look to see this is the case:
more .git/refs/heads/master
more .git/refs/heads/add-more-sessions

But what happens if the two branches diverge?  Let’s give it a try.

First lets change the master branch:

echo “Aurelia” >> presentations.txt
git add -A
git commit -m “add Aurelia"
<*******************   Slide **************************>
Next lets change add-more-sessions branch:
git checkout add-sessions
cd codecamp
echo “DDD” >> sessions.txt
git add -A
git commit -m “add ddd"
<*******************   Slide **************************>
Now we have two branches, each with different commits that change files.  What happens when we merge these two branches?
Git first finds a common ancestor.  In this case the commit both the branches started on.   It then determines which trees and files have changed between this base commit and master and then with add-more-sessions branch.  Since all of this content is hashed, figuring out if a something is different is pretty fast and efficient.  It then takes any differences it finds, merges the files (automatically if it can), creates the new objects based on this merge (trees and objects) and creates a merge commit.  Let’s give it a try:

git checkout master
git merge add-sessions

<*******************   Slide **************************>
Notice this time we get prompted to add a commit message.  That’s because it’s not a fast forward and git needs to make a new commit to record our new “snapshot” of our data.

Who is the parent of this merge commit?
This commit has two parents, master and add-more-sessions.

Let’s look at the commit:
git log
git cat-file -p <sha>

It’s important to remember that this commit is again pointing to a tree that is a new snapshot of your application.  It contains new objects for any files that had to be merged between D and E.

Obviously sometimes there are conflicts and git needs your help resolving these.  You can resolve these merges by using a 3-way merge tool such as kdiff or  you can do it by hand just by using the comments added to the file by git.

Now let’s look at another way of merging called rebasing.  We’re going to back up and re-merge those changes:

let’s revert this merge
git reset —hard <sha> of Aurelia
<*******************   Slide **************************>

Let’s look at the sha of D before we rebase.
cat .git/refs/heads/master

git checkout master
git rebase add-sessions
Look at sha of D again:

'''
cat .git/refs/heads/master
'''

It's a different SHA!  D'
