You have encountered a basic stumbling block for those who try to use Git without a tutorial, or with a bad tutorial (of which there are, unfortunately, many). Your mistake here is thinking that "merge", in Git, involves two versions of some file. This is not the case! Merging in Git is on a commit basis and involves three versions of each file.

Before we can get into merging, we have to start with commits. Without the proper base,1 merging won't make any sense. So let's jump into that first. Note that you may want to read this twice before you tackle your merge.

1There's a pun here that will fly right over your head if you're not already familiar with the ideas behind merging. 😀
Commit = snapshot + metadata

Git is, at its heart, all about commits. Git is not really about files or branches. It's true that each commit stores files, and we (or Git at least) organize commits into branches, using branch names to help us find individual commits. But it's the commit itself—or the collection of commits—that's the heart of the repository.

Git stores these commits in an object database. We won't go into most of the details here, though we will skim a few necessary parts, but if you are curious, you can peek inside the hidden .git folder, where you will find a sub-folder named objects. Inside this are potentially many more folders, which store the objects in various forms ("loose" and "packed" but you don't need to care about this here). Each object, including each commit, is numbered, with a unique ID that is expressed in hexadecimal, such as d420dda0576340909c3faff364cfbd1485f70376. (This particular one is a commit in the Git repository for Git.)

This hash ID is the "true name" of the internal object, and Git literally requires it to find the object in the objects database. These names are not friendly to humans, though, so Git provides a separate secondary database—one that's not nearly as well organized, nor as well implemented, really—that stores names: branch names, tag names, and all other kinds of names. Each of these names simply stores one hash ID, which is in fact all that's necessary.

So, when you use a branch name like main or feature or whatever, you're really providing Git with a raw hash ID, that's been hidden behind this name. It's worth running git rev-parse a few times to get a feel for this mechanism:

git rev-parse main

might produce:

d420dda0576340909c3faff364cfbd1485f70376

for instance, if you have a clone of the Git repository for Git (though by now its main/master has moved on; I haven't updated my clone in more than a week now for home-life reasons).

When you clone a Git repository, it's the underlying objects that you are copying. The names in your names database are yours, not the original clone's, but the objects, which are all strictly read-only once they're created, get shared. You get a copy but you (and your Git software) are forbidden from changing any of these. Not even Git can change a Git commit. You just add new commits to the repository. That's how history exists: the existing commits continue to exist. They're just now in two repositories: the original, and your clone. Make more clones, and you make yet more copies of the objects.

With that said, let's look at the anatomy of one particular commit, in this case d420dda0576340909c3faff364cfbd1485f70376:

$ git cat-file -p d420dda0576340909c3faff364cfbd1485f70376 | sed 's/@/ /'
tree 13b45e4ccc34572dce66dc79468b66c0b383a560
parent c68bd3ec22a1afc85b0b897834b2524aedbd0553
author Junio C Hamano <gitster pobox.com> 1665507772 -0700
committer Junio C Hamano <gitster pobox.com> 1665509772 -0700

The second batch

Signed-off-by: Junio C Hamano <gitster pobox.com>

That, right there, is the entire commit object, as seen in every Git clone of the Git repository for Git. You can see that commits are pretty small! But each one has a tree object: that first line, tree followed by an internal object ID, is required in every commit.

The tree object in the commit represents a permanent archive of every file. This is another object (and you can git cat-file -p it, if you like, to see how it works in great detail), but what we see in the output above is the metadata for the commit:

    the commit has a parent, with a raw hash ID: this is another commit object;
    the commit has an author and a committer: these are text strings giving the name and email address of the person who made the commit, along with some date-and-time stamps; and
    the commit has a log message, which is what you see when you run git log.

The git log command uses the stored hash ID here to find the previous commit. Most commits have exactly one parent line, but a few have mroe than one parent, making them merge commits, and at least one commit in any non-empty repository has no parent because it was the first commit.

We say that the commit points to its parent or parents, and if we use single uppercase letters to stand in for real hash IDs—which are big, ugly, and random-looking2—we get drawings that look like this:

... <-F <-G <-H   <-- main

Here, the name main, a branch name, points to (contains the hash ID of) commit H. Commit H itself contains, indirectly, a full snapshot of every file—that's the tree <hash> line—and directly contains metadata, including a parent line. So commit H points to earlier commit G.

Commit G, being a commit, contains a snapshot and metadata, so G points to earlier commit F. Commit F is a commit, so it points to a still-earlier commit, which points back to an even-earlier commit, and so on, backwards, down the line to the very first commit ever (presumably commit A in our drawing).

2They're actually not random at all, and concretely, the hash ID d420dda0576340909c3faff364cfbd1485f70376 is simply the SHA-1 checksum of the content of the above commit, except that this content is prefixed by commit 284 and an ASCII NUL byte, with 284 being the decimalized size of the rest of the object. The fact that the previous hash IDs on the parent lines and the date-and-time-stamps are themselves unique means that the new hash ID is unique.3

3Anyone familiar with the pigeonhole principle should immediately object here. That objection is correct and means that Git will eventually fail. You can calculate the probability of failure with a fancy formula, and it turns out to be vanishingly small until the objects database holds more than about 1.7×1015 objects, at which point it starts to creep up towards the probability of undetected disk-drive errors. We live with those; we can live with SHA-1 collisions. Even so, Git is slowly moving towards SHA-256.
A special feature of a branch name

We're going to skip a lot here and just look at one special feature of branch names. We already know that the name points to a commit. We can also draw this:

...--G--H   <-- main, feature

where we have a single commit, H, that is pointed-to by more than one branch name. When this is the case, all the commits up to and including H are on both branches. Checking out either branch, with git switch main or git switch feature, gets us the files from commit H. But, as a special feature, checking out that name "attaches" the special name HEAD to that name:

...--G--H   <-- main (HEAD), feature

Here, we're on commit H and branch main. The files we have available to us are those from commit H. If we now run:

git switch feature

the picture changes slightly:

...--G--H   <-- main, feature (HEAD)

We're still on commit H, but we're "on" it through the name feature. We have the same files, but something wacky is about to happen.

Let's make a new commit now, in the usual way Git has for making commits (we modify some files and git add and git commit). We get a new commit, which gets a new, unique ID; we'll just call this I and draw it in:

          I
         /
...--G--H

