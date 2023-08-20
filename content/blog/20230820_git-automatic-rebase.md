---
title: 'Automatic Rebase of Last Commit in Git'
date: 2023-08-20T09:00:00-00:00
draft: false
---

## Introduction

I often find myself in the situation where I have just pushed a branch with my changes and created a pull request, only to later realize that I forgot
to add a file or made a typo in my code. In such cases, my goal is to make a small change while maintaining a clean Git history, so I opt to rebase
the last commit to essentially hide it. Of course this wouldn't work if there was a chance that someone already used my branch or I cooperated with
someone on it. But usually the branch is just mine and I make the changes almost immediately after pushing it so I think it's worth it to keep the
history clean.

## Manual Procedure

Over the years, I've developed a manual procedure that I can execute quite swiftly. Essentially, it involves creating a commit with any message,
running a rebase on `HEAD~2`, marking the commit as `fixup` in Vim, and finally performing a force push. It was just a matter of few seconds. However,
once I wondered whether I could save myself a few keystrokes and automate it.

## Automated Procedure

In fact, you can. While there might be other methods, I decided to strictly automate my procedure, and I was surprised to discover that I
could automate Vim 🤯:

```fish
function git_rebase_last_and_push
    set -lx GIT_EDITOR "vim +2 -c 'normal cwf' -c 'wq'"
    git commit -a -m "wip"
    git rebase -i HEAD~2
    git push -f origin HEAD
end
```

The trick is in setting the `GIT_EDITOR` environment variable. It tells Git to use Vim as the editor and to open the file at line 2 (which is exactly
the line with my last commit). Then it runs 2 Vim commands. First `normal cwf` which is to `c`hange the `w`ord under the cursor to `f` (as
in `fixup`). And second `wq` which is to `w`rite the file and `q`uit Vim. And that's it.

So when you run `git_rebase_last_and_push` in your shell (Fish shell in my case) it will commit everything with "wip" message, it will start the
rebase process with the last two commits which will trigger Vim, it will fixup the last commit and quit and finally everything is force pushed to
the remote repository. The force push is necessary because the commit history was changed. This is the part that's a bit dangerous and does not work
if you cooperate on the code with others but as long as the branch is just yours I encourage you to keep the history clean.

## Conclusion

It's quite satisfying to see it in action. I hope you will find it useful as well.
