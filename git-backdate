#!/usr/bin/env python3
"""Backdate a commit or range of commit to a date or range of dates.
"""
import argparse
import datetime as dt
import math
import os
import random
import re
import sys
from pathlib import Path
from subprocess import call, check_call, check_output


def get_commits(commitish: str) -> list[str]:
    """If commitish is a range, return a list of commits in that range."""
    if ".." not in commitish:
        commitish = f"{commitish}..HEAD"
    result = check_output(["git", "rev-list", commitish]).splitlines()
    # git rev-list returns commits in reverse chronological order
    # we don't really care atm, but it does make debugging awkward.
    return [c.decode() for c in result[::-1]]


def _parse_date(dateish: str) -> dt.date:
    """Parse a dateish string into a datetime object."""
    if not re.match(r"\d{4}-\d{2}-\d{2}", dateish):
        dateish = (
            check_output(["date", "--iso-8601", "--date", dateish]).strip().decode()
        )
    return dt.datetime.strptime(dateish, "%Y-%m-%d").date()


def get_dates(dateish: str) -> list[dt.date]:
    """Dateish is either one or two dateish strings, separated by .. if there are two.
    A dateish string can be an ISO 8601 date or anything parsed by your system's `date`,
    which often includes things like "last month" and "3 days ago"."""
    if ".." in dateish:
        return [_parse_date(d) for d in dateish.split("..")]
    result = _parse_date(dateish)
    return [result, result]


def _get_timestamp(
    date: dt.date, min_hour: int, max_hour: int, greater_than: dt.datetime | None = None
) -> dt.datetime:
    """Return a random timestamp on the given date, between min_hour and max_hour."""
    greater_than = greater_than or dt.datetime.combine(date, dt.time.min)
    min_timestamp = dt.datetime.combine(date, dt.time(min_hour))
    max_timestamp = dt.datetime.combine(date, dt.time(max_hour, 59))
    now_timestamp = dt.datetime.now()
    min_timestamp = min_timestamp if min_timestamp > greater_than else greater_than
    max_timestamp = max_timestamp if max_timestamp < now_timestamp else now_timestamp
    interval = int((max_timestamp - min_timestamp).total_seconds())
    return min_timestamp + dt.timedelta(seconds=random.randint(0, interval))


def rebase_in_progress() -> bool:
    """Return True if a rebase is in progress."""
    git_root = Path(
        check_output(["git", "rev-parse", "--show-toplevel"]).strip().decode()
    )
    return any(
        (git_root / ".git" / f).exists() for f in ("rebase-merge", "rebase-apply")
    )


def rewrite_history(
    commits: list[str],
    start: dt.date,
    end: dt.date,
    business_hours: bool,
    no_business_hours: bool,
) -> None:
    days = [start + dt.timedelta(days=day) for day in range((end - start).days + 1)]
    min_hour = 0
    max_hour = 23
    if business_hours:
        days = [day for day in days if day.weekday() < 5]
        min_hour = 9
        max_hour = 17
    elif no_business_hours:
        min_hour = 18
        max_hour = 23
    duration = len(days)
    commits_per_day = math.ceil(len(commits) / duration)
    last_timestamp = None
    commit_count = len(commits)
    day_progress = 0
    for commit_index in range(commit_count):
        progress = (commit_index + 1) / commit_count
        # first, choose the date
        date_index = round(progress * (duration - 1))
        date = days[date_index]
        if not last_timestamp or date == last_timestamp.date():
            day_progress += 1
        else:
            day_progress = 0

        # if we only have one commit per day at most, we can use the whole day.
        # otherwise, we need to limit the time range further to avoid collisions.
        if commits_per_day <= 1:
            _max_hour = max_hour
        else:
            _max_hour = min_hour + int(
                (max_hour - min_hour) * (day_progress / commits_per_day)
            )

        timestamp = _get_timestamp(
            date, min_hour=min_hour, max_hour=_max_hour, greater_than=last_timestamp
        )

        # Set both the author and committer dates
        check_call(
            [
                "git",
                "commit",
                "--amend",
                "--date",
                timestamp.isoformat(),
                "--no-edit",
            ],
            env=dict(os.environ, GIT_COMMITTER_DATE=timestamp.isoformat()),
        )
        last_timestamp = timestamp
        check_call(["git", "rebase", "--continue"])


def normalize_commit(commit: str) -> str:
    return check_output(["git", "rev-parse", commit]).strip().decode()


def is_equal(commitish_a: str, commitish_b: str) -> bool:
    return normalize_commit(commitish_a) == normalize_commit(commitish_b)


def main() -> None:
    parser = argparse.ArgumentParser(description=sys.modules[__name__].__doc__)
    parser.add_argument(
        "commits",
        metavar="COMMITS",
        default="HEAD",
        help="COMMITS is a commit or range of commits to backdate. If only one commit is given, it will be used as the end of the range.",
    )
    parser.add_argument(
        "dates",
        metavar="DATES",
        default="now",
        help='DATES is a date or range of dates to backdate to, can use human readable dates. Separate by .., eg "1 week ago..yesterday"',
    )
    parser.add_argument(
        "--business-hours",
        action="store_true",
        help="Backdate to business hours: Mon–Fri, 9–17",
        default=False,
    )
    parser.add_argument(
        "--no-business-hours",
        action="store_true",
        help="Backdate to outside business hours: 19–23, every day of the week",
        default=False,
    )
    args = parser.parse_args()

    if args.business_hours and args.no_business_hours:
        print("Cannot use both business hours and outside business hours")
        sys.exit(1)

    commits = [normalize_commit(c) for c in get_commits(args.commits)]
    start, end = get_dates(args.dates)

    if not commits:
        print("No commits found")
        sys.exit(1)

    # Make sure our current commit sits on top of the commit range with --is-ancestor
    for commit in commits:
        is_head = is_equal(commit, "HEAD")
        is_ancestor = call(["git", "merge-base", "--is-ancestor", "HEAD", commit])
        if not is_head and not is_ancestor:
            print(f"Current commit is not an ancestor of the commit range {commit}")
            sys.exit(1)

    # We construct a sed command to change only our commits
    short_commits = [c[:7] for c in commits]
    sed_command = rf"sed -i.bak -E 's/^pick ({'|'.join(short_commits)})/edit \1/'"
    check_call(
        ["git", "rebase", "-i", commits[0] + "^"],
        env=dict(os.environ, GIT_SEQUENCE_EDITOR=sed_command),
    )

    # Global try/except to make sure we reset the repo if we fail
    try:
        rewrite_history(
            commits,
            start,
            end,  # we want to include the end date
            business_hours=args.business_hours,
            no_business_hours=args.no_business_hours,
        )
    except Exception:
        if rebase_in_progress():
            check_call(["git", "rebase", "--abort"])
        raise
    finally:
        # Shouldn't happen, but let's make sure we end on a clean repo
        if rebase_in_progress():
            check_call(["git", "rebase", "--continue"])


if __name__ == "__main__":
    main()
