+++
title = "Quick Git tip"
date = 2025-10-20T11:34:00+02:00
tags = ["git"]
draft = false
+++

Recently, one of the groups in my institute changed the base URL of their Git repository. This was done to mark a clear distinction in the release of a new software version. All projects were migrated to this new URL, which gives developers two possible solutions:

-   Re-clone all projects using the new URL
-   Update the URL value in the `.git/config` file of every affected repository

On my side, I use `insteadOf` directives for Git URLs that I frequently clone from. These directives allow URL aliases to be defined within the context of Git. For example to refer to `https://gitlab.com/thatRepo/`, one could define the alias `thatRepo` by running the command

```shell
git config --global url."https://gitlab.com/thatRepo".insteadOf "thatRepo:"
```

Which will add the following lines to your `.gitconfig` file:

> [url "<https://gitlab.com/thatRepo/>"]
>
> insteadOf = thatRepo:

then if I need to clone a specific project from that repository, I can simply type

```shell
git clone thatRepo:thatProject
```

When the time came to migrate to the group's new repository, I simply updated the URL value my `.gitconfig`.
