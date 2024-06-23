# Use Git to Rewrite History

This lab gives you more hands-on time with Git. You will check out specific commits, tag them, revert, and roll back changes.

### Create a Local Git Repository

```bash
mkdir git_lab
cd $_
git init .
```

### Create `Hello.java` Containing the Following

```java
public class Hello {
    public static void main(String[] argv) {
        System.out.println("Hello, World");
    }
}
```

### Add the File to the Repository and Commit It

```bash
git add Hello.java
git commit -m "Initial commit"
```

### Change the “Hello, World” Program

```java
public class Hello {
    public static void main(String[] argv) {
        System.out.println("Hello, " + argv[0] + "!");
    }
}
```

### Commit Your Change

```bash
git commit -am "Update Hello.java to greet by name"
```

### Add a Default Value if a Command Line Argument is Not Supplied

```java
public class Hello {
    public static void main(String[] argv) {
        String name = "World";
        if (argv.length != 0) {
            name = argv[0];
        }
        System.out.println("Hello, " + name + "!");
    }
}
```

### Commit the Change

```bash
git commit -am "Add default value for greeting"
```

### View the History with `git log`

Using `git log`, we can see the history of our project, including all of our previous commits. This is extremely useful because it allows us to revert our code to previous versions.

```bash
git log
```

As you can see in the output, we have a commit for each of the changes we've made in the repository.

### Customize `git log` Output

The `git log` command is extremely customizable. You can define what is shown in the output. Try some of these examples:

```bash
git log --pretty=oneline
git log --pretty=oneline --max-count=2
git log --pretty=oneline --since='5 minutes ago'
git log --pretty=oneline --until='5 minutes ago'
git log --pretty=oneline --author="<your name>"
git log --pretty=oneline --all
```

You can also specify the range of commits to show:

```bash
git log --all --pretty=format:'%h %cd %s (%an)' --since='7 days ago'
```

This command displays pertinent information without overloading the screen:

```bash
git log --pretty=format:'%h %ad | %s%d [%an]' --graph --date=short
```

#### Details of the Format:

- `--pretty="..."` defines the format of the output.
- `%h` is the abbreviated hash of the commit.
- `%d` are any decorations on that commit (e.g., branch heads or tags).
- `%ad` is the author date.
- `%s` is the comment.
- `%an` is the author name.
- `--graph` informs Git to display the commit tree in an ASCII graph layout.
- `--date=short` keeps the date format nice and short.

### Git Aliases

Git supports aliases, so you don’t have to type the entire string. Here are some helpful aliases I use. To make life easier, add them to `~/.gitconfig`.

```ini
[alias]
  co = checkout
  ci = commit
  st = status
  br = branch
  hist = log --pretty=format:'%h %ad | %s%d [%an]' --graph --date=short
  type = cat-file -t
  dump = cat-file -p
```

Now, when you want to see the logs in that specific format, you can type `git hist`.

### Checkout Any Previous Snapshot

Suppose you want to revert your current directory back to a previously committed version. Going back in history is very easy. The `git checkout` command will copy any snapshot from a previous commit in the repository to the working directory.

#### Get the Hashes for Previous Versions

In Git, committed versions are uniquely identified with a hash code. This is the strange sequence you saw associated with each commit (something like `commit 56ad4d905ead7db10dde52518b4b161281c0f4ff`) in `git log`. Let’s take a closer look.

```bash
git log
```

Examine the log output and find the hash for the first commit (at the bottom). Use that hash code (the first seven characters are enough) in the command below. By default, `git log` lists the commits in reverse chronological order (look at the timestamps). Once you find the hash for the first commit, check the contents of the `Hello.java` file.

```bash
git checkout <hash>
```

You will see something similar to:

```
Note: checking out '9416416'.

You are in 'detached HEAD' state. You can look around, make experimental
changes and commit them, and you can discard any commits you make in this
state without impacting any branches by performing another checkout.

If you want to create a new branch to retain commits you create, you may
do so (now or later) by using -b with the checkout command again. Example:

  git checkout -b new_branch_name

HEAD is now at 9416416... First Commit
```

A “detached HEAD” message in Git means that HEAD (the part of Git that tracks what your current working directory should match) points directly to a commit rather than a branch. Any changes committed in this state are only remembered as long as you don’t switch to a different branch. When you check out a new branch or tag, the detached commits will be “lost” (because HEAD has moved). If you want to save commits made in a detached state, you must create a branch to remember the commits.

#### Confirm the File Has Been Reverted

```bash
cat Hello.java
```

#### Return to the Latest Version

```bash
git checkout main
```

### Tag Commits with Names for Future Reference

You can create tags for each commit if you don’t like the default names. Let’s call the current version of the hello program version 1 (v1).

#### Tagging Version 1

```bash
git tag v1
```

Now you can refer to the current version of the program as v1.

#### Tagging Previous Versions

Let’s tag the version immediately before the current version as v1-beta. First, we need to check out the previous version. Rather than look up the hash, we will use the `^` notation to indicate “the parent commit of v1”.

If the `v1^` notation gives you any trouble, you can also try `v1~1`, which references the same version. This notation means “the first ancestor of v1”.

```bash
git checkout v1^
cat Hello.java
```

You will be in a detached HEAD state and have the previous version of the file. Tag the commit with `v1-beta`.

```bash
git tag v1-beta
```

#### Checking Out by Tag Name

Now try going back and forth between the two tagged versions.

```bash
git checkout v1
git checkout v1-beta
```

To view all tags, use the `git tag` command.

### Undo Local Changes (Before Staging)

#### Checkout Main

Make sure you are on the latest commit in the main branch before proceeding.

#### Change `Hello.java`

Sometimes you have modified a file in your local working directory and wish to revert it to what has already been committed. The `checkout` command will handle that.

Change `Hello.java` to have a bad comment.

```java
public class Hello {
    public static void main(String[] argv) {
        // This is a bad comment. We want to revert it.
        String name = "World";
        if (argv.length != 0) {
            name = argv[0];
        }
        System.out.println("Hello, " + name + "!");
    }
}
```

#### Check the Status

First, check the status of the working directory using `git status`.

We see that the `Hello.java` file has been modified, but the changes haven’t been staged yet.

#### Revert the Changes in the Working Directory

Use the `checkout` command to check out the repository’s version of the `Hello.java` file.

```bash
git checkout Hello.java
git status
cat Hello.java
```

The `status` command shows us that there are no outstanding changes in the working directory. The “bad comment” is no longer part of the file contents since we “checked out” the file to its previous version.

### Undo Staged Changes (Before Committing)

We just undid unstaged changes; now, let’s try undoing staged changes.

#### Change the File and Stage the Change

Modify the `Hello.java` file to have a bad comment.

```java
public class Hello {
    public static void main(String[] argv) {
        // This is an unwanted but staged comment
        String name = "World";
        if (argv.length != 0) {
            name = argv[0];
        }
        System.out.println("Hello, " + name + "!");
    }
}
```

And then go ahead and stage it.

```bash
git add Hello.java
```

#### Check the Status

Check the status of your unwanted change.

The status output shows that the change has been staged and is ready to be committed.

#### Reset the Staging Area

Fortunately, the status output tells us exactly