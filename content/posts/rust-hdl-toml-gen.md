+++
title = "Rust HDL TOML generator"
date = 2025-02-15T14:34:00+01:00
tags = ["emacs", "vhdl"]
draft = false
+++

I use [VHDL-LS](https://github.com/VHDL-LS/rust_hdl) as my VHDL language server for [lsp mode](https://emacs-lsp.github.io/lsp-mode/). This combined with [vhdl major mode](https://www.gnu.org/software/emacs/manual/html_mono/vhdl-mode.html), and other packages such as [vhdl-ext](https://github.com/gmlarumbe/vhdl-ext), provides me with a very robust IDE within Emacs for my VHDL projects.

A requirement of the language server is to have a `vhdl_ls.toml` file which includes all sources used on a project. Even with glob patterns, it can get annoying when a new source/folder/module needs to be added; the `toml` file must be updated accordingly and one can easily forget to add that one folder or file.

To address this, I wrote a [VHDL-LS TOML generator](https://github.com/bjfer/hdl-toml) to assist in generate and maintain these toml files. It is a simple `elisp script` that should be added to a folder in Emacs' load-path and then, in the init file, just add the line `(require 'hdl-toml)`. It can create projects, add, update or delete sources and provides some customization, namely the usage of absolute of relative paths, glob patterns for sources and which vhdl standard do use.
