# git-backdate

git-backdate helps you to change the date of one or multiple commits to a new date or
a range of dates. 

## Features

- Understands business hours, and that you might want to have your commits placed inside or outside of them.
- Commits are randomly distributed within the given time window, while retaining their order. No 12:34:56 for you
  anymore!
- Given a single commit, git-backdate will automatically assume that you want to rebase the entire range from there to
  your current `HEAD`.
- Sets author date and committer date so you don't have to look up how to set both of them every time you fudge a commit
  timestamp.
- Python, but with near-zero dependencies (see below; only `sed` and `date`), so you can just download and run it
  without Python package management making you sad.


## Usage

Backdate all your unpushed commits to the last three days, and only during business hours:

```shell
git backdate origin/main "3 days ago..today" --business-hours
```

Backdate only some commits to have happened outside of business hours:

```shell
git backdate 11abe2..3d13f 2023-07-10 --no-business-hours
```

Backdate only the current commit to a human readable time:

```shell
git backdate HEAD "5 hours ago"
```

## Installation

Drop the `git-backdate` file somewhere in your `PATH` or whereever you like:

```shell
curl https://raw.githubusercontent.com/rixx/git-backdate/main/git-backdate > git-backdate
chmod +x git-backdate
```

The magic of git will now let you use `git backdate` as a command.


### Requirements

`git-backdate` tries to only require git and Python. However, it also relies on

- `sed` if you want to backdate more than the most recent commit <sup>for perfectly fine reasons, don't worry about it</sup>
- `date` if you want to pass date descriptions like "last Friday" or "2 weeks ago"

## … why.

I started various versions of this in uni, when I did my assignments last minute and wanted to appear as if I had my
life together. I still sometimes use it like that (especially to make the 3am commits look less deranged), but there
have been new and surprising use cases. Most of these have been contributed by friends and do not reflect on me, nor do
they represent legal or career advice:

- Did work things outside work hours, but don't want to nudge your workplace culture into everybody working at all times.
- Worked an entire weekend for a client, but don't want them to get used to it and start calling you on every weekend.
- Made some fixes during a boring meeting, but pretended to pay attention throughout.
- <your reason here, please share with the class>

## Caveats

Commit dates are part of a commit's metadata. Changing a commit's date changes its hash.
You very likely only want to run `git backdate` on commits that you have not pushed yet,
because otherwise you would have to `--force` push your new history. I know, I know,
you're using `--force-with-lease`, so you won't destroy data, but your collaborators
or integrations will still be somewhat miffed.

Also, obviously, use with care and compassion.
