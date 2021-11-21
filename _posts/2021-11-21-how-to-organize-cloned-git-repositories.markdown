---
layout: post
title:  "How to organize cloned Git repositories"
date:   2021-11-21 15:04:20 +0200
categories: git
---
Working with multiple Git repositories in various contexts can get tricky. Some of the repositories may be work related, others personal, or for a customer. I will share a few tips about how to structure the repositories cloned locally on your machine.

## Choosing the directory structure

- `~/git/` - the top-level directory has a short name. It is immediately clear what the contents is - anything that resides in Git. Having the directory in your `$HOME` path makes it easily accessible.
- `~/git/personal`, `~/git/company1`, `~/git/company2` - the sub-directories provide additional context about the repositories being cloned. This structure makes it easy to specify the `user.email` and `user.name` for all repos there.
- You may nest the directory structure further, e.g. if you have multiple sub-projects. Be careful with nesting, or you will end up with many directories to navigate.

## Provide the configuration per context

Did you ever commit to a Git repository using the wrong email? The approach of doing `git config --local user.email` per repository is error prone. It's easy to forget updating config, especially in a workflow where you frequently clone repositories.

A better approach would be to include a per-context configuration within the `.gitconfig`.

{% highlight conf %}
[includeIf "gitdir:~/git/personal/"]
    path = .gitconfig-personal
[includeIf "gitdir:~/git/example/"]
    path = .gitconfig-example
{% endhighlight %}

Then specify the user configuration for the personal context separately within the `~/.gitconfig-personal` file.

{% highlight conf %}
[user]
    email = jordan.jambazov@gmail.com
    name = Jordan Jambazov
{% endhighlight %}

## Don't hesitate to delete stale clones

Many repositories are being cloned just for a quick one-time adjustment. Once you observe a repository that is not being actively used in your daily operations - don't hesitate to delete it. Of course, make sure all changes have been pushed to the remote.

Maybe you'll need it someday? No worries, then you'll clone it again.

## Conclusion

Chosing a strategy to organize your cloned Git repositories is always subjective. It depends on your workflow and habits. The proposed approach works well when you have multiple contexts. Feel free to adapt it to your needs.
