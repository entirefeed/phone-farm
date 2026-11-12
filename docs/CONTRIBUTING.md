# Contributing

Thank you for helping. Open a pull request or an issue.

## What gets added

1. **Open source, or at least free to read.** Link the repository. If part of the tool is closed, say so in the description.
2. **In scope.** It connects to phones, shows or controls their screens, automates their apps, manages a fleet, schedules jobs, or monitors a rack.
3. **Described honestly.** Say in the pull request whether you have run the tool and on what. An entry from someone who has run it is worth more than one copied from a README.

## What does not get added

- Engagement bots, account creators and follower tools.
- Tools whose purpose is to hide automation from a platform or to fake a device's identity.
- Paid products with no open-source part. Hosted services are mentioned only as alternatives to owning phones.

## How to write an entry

One table row:

```text
| [owner/name](https://github.com/owner/name) | What it does, in one or two plain sentences. Where it falls short. | Licence | Stars | Last push |
```

- Say what the tool does, then what it does not do well. "Built for testing one app, not for running a fleet" helps a reader more than praise.
- No adjectives such as "powerful" or "best".
- If the repository is archived or has had no push for a year, say so.

## Getting the numbers

Read them from the GitHub API on the day you open the pull request. Do not type them from memory.

```bash
gh api repos/OWNER/NAME --jq '[.full_name, .stargazers_count, (.license.spdx_id // "none"), .pushed_at[:10], .archived] | @tsv'
```

Round stars above 1,000 to one decimal (8,499 becomes 8.5k). If the API reports the licence as `NOASSERTION`, open the licence file, write what it says and mark it with `*`.

## Refreshing the list

Stars and dates go stale. Anyone may open a pull request that refreshes every row with the command above and changes the date at the top of the list. Archived tools move to the "archived" line of their section.
