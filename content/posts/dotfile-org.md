+++
title = "Dotfiles with Org Babel and Host-Specific Configuration"
date = 2026-08-14T00:37:00+02:00
tags = ["emacs", "org"]
draft = false
+++

Like many Emacs users, I use `org-babel-load-file` to load custom Emacs Lisp source blocks from an Org file ([as described here](https://orgmode.org/worg/org-contrib/babel/intro.html#emacs-initialization)). I also use Org Babel’s tangling support to maintain separate Org files for several dotfiles, such as `~/.bashrc`, `~/.gitconfig`, `~/.signature`, and `~/.ssh/config`. This helps me keep my setup consistent across work and home machines.

I use the same approach for software that my Emacs setup depends on. For example, I compile mu/mu4e from source and keep the installation steps in a single Org Babel source block. That makes it easy to update or reinstall by simply re-running the relevant block.

Many online tutorials cover Org Babel and tangling, but few show how to use Org’s broader feature set to handle scenarios such as:

-   Reusing the same information across multiple code blocks to avoid typos, such as a full name or work email address
-   Managing dependencies between Org files
-   Defining reusable functions for common tasks, such as downloading custom `.el` files into a `site-lisp` directory
-   Maintaining host-specific configuration, such as locally generated SSH keys or GPG subkeys for Git signing and authentication

To support this, I wrote an `env.org` file that contains the information and helper functions I need to generate both my dotfiles and my Emacs init file, while keeping them consistent with the target host.


## Reusing information {#reusing-information}

For static information, I define named blocks and reference them using Org’s [noweb syntax](https://orgmode.org/manual/Noweb-Reference-Syntax.html). For example, the following block contains my name:

```text
#+NAME: full-name
: Bruno Fernandes
```

I can then use it in both my Emacs init Org file:

```emacs-lisp
;; emacs-lisp block
(setq user-full-name "<<env.org:full-name()>>")
```

as my `signature.org` file:

```tesxt
Mit freundlichen Grüßen / Best regards,

--
<<env.org:full-name()>>
```

The same approach also works for host-specific values. For example:

```text
#+NAME: host-username
#+begin_src emacs-lisp
(user-login-name)
#+end_src
```

This retrieves the current host’s username, which I can then use in a Docker installation block:

```shell
# Add user to docker group (needs logout)
groupadd docker
usermod -aG docker <<env.org:host-username()>>
```


## Short names, full paths {#short-names-full-paths}

Whenever I need a file or directory location, I always use the full path rather than relying on assumptions. Since most of my files live somewhere under my home directory, I wrote a helper function that expands short paths into full ones:

```text
#+NAME: expand-home-folder
#+begin_src emacs-lisp :var path=""
;; Full path of location (defaults to home)
(expand-file-name (concat "~/" path))
#+end_src
```

For example, to enable [Git prompt support in Bash](https://git-scm.com/book/en/v2/Appendix-A:-Git-in-Other-Environments-Git-in-Bash), I need to download a script from the Git repository and source it from `.bashrc`. The following block returns the full path where I want the file to live:

```text
#+NAME: git-prompt-file
#+CALL: expand-home-folder(".gitprompt.sh")
```

I can use that short name as an argument to a helper function that wraps `wget` and expects a URL and a destination path:

```text
#+CALL: wget-file("https://raw.githubusercontent.com/git/git/master/contrib/completion/git-prompt.sh", git-prompt-file())
```

And I can reuse the same reference in my `bash.org` file:

```text
#+begin_src shell
# Git prompt
source <<git-prompt-file()>>
#+end_src
```

This keeps paths consistent across installation and configuration steps. I use the same approach for Git repositories that I clone frequently and for software that I compile from source, such as mu:

```text
#+begin_src shell :var git-link=https://github.com/djcb/mu.git folder=mu
;; Note that var must have the same name as the one defined in git_clone
<<git_clone>>
cd mu
meson setup build -Demacs=<<bash.org:emacs-bin-location()>>emacs -Dlispdir=<<emacs.org:emacs-lisp-dir()>>
meson compile -C build
meson install -C build
#+end_src
```

This example also shows how one Org file can depend on values defined in another. Building mu/mu4e requires the Emacs binary location and the target Lisp directory, while Emacs itself must later know where to find the installed mu4e Lisp files.


## Host specific information {#host-specific-information}

I keep an alist of all proprieties which are host specific.

```text
#+NAME: host-info
#+begin_src emacs-lisp
''(("<<work-host()>>" :gpg-sign 111AAABBB :gpg-auth CCCDDD555 :email <<work-email()>> :gitlab-key /common/user123/.ssh/gitlab_key)
("<<my-laptop()>>" :gpg-sign EEEFFF0000 :gpg-auth 123456 :email <<my-email()>> :gitlab-key /home/bjfer/.ssh/gitlab_012026 :github-key /home/bjfer/.ssh/github_022025))
#+end_src
```

Since the hostname is available through `(system-name)`, I wrote a helper function that looks up the appropriate value for the current host:

```text
#+NAME: get-host-info
#+begin_src emacs-lisp :var property='info
(plist-get (cdr (assoc (system-name) <<host-info()>>)) property)
#+end_src
```

I can then use it in blocks that tangle host-specific values, for example in `gitconfig.org`:

```text
[user]
signingKey = <<env.org:get-host-info(':gpg-sign)>>
```

I can also use it to conditionally tangle sections.

```text
#+NAME: tangle-host
#+begin_src emacs-lisp :var property='nil path='nil
(if <<get-host-info>> (expand-file-name (symbol-name path)) '"no")
#+end_src
```

In my `sshconfig.org` file, the following section will not be tangled on my work machine, because no GitHub SSH key is defined there:

```text
#+begin_src text :tangle (org-babel-ref-resolve "env.org:tangle-host(':github-key,'~/.ssh/config)")
Host github
User git
Hostname github.com
IdentityFile <<env.org:get-host-info(':github-key)>>
#+end_src
```

I’m sure there are more _Elisp-y_ ways to do this, but my goal was simply to show how Org Babel features, combined with a bit of Emacs Lisp, can make this setup a flexible system for managing configuration files.
