+++
title = "A Git pre-commit hook for VHDL projects"
date = 2023-11-02T22:47:00+01:00
tags = ["git", "vhdl"]
draft = false
+++

For a while, I have been considering doing a series of Youtube videos inspired by the awesome series from Rainer König about [Org Mode](https://youtube.com/playlist?list=PLVtKhBrRV_ZkPnBtt_TD1Cs9PJlU0IIdE&feature=shared), just to show how Emacs is the best free VHDL editor. Far too often, I see experienced HDL developers using the default editor provided by the FPGA vendor tools. You don't need to dive deep into Emacs to take full advantage of the [VHDL mode](https://www.gnu.org/software/emacs/manual/html_mono/vhdl-mode.html) available in the editor.

As an incentive, here is the git pre-commit hook I use in most of my VHDL projects. The hook does two things:

-   checks if there are new HDL sources that have not been added to the repository
-   for all modified HDL sources, use Emacs in batch mode to align the code and remove unnecessary whitespace

The most common whitespace I add is trailing and extra blank lines. Emacs has a straightforward command to delete the former, `delete-trailing-whitespace`, and using a `replace-regexp`, I can substitute multiple blank lines for just one. Aligning VHDL code is done with the `vhdl-beautify-buffer` command.

The prerequisite to running this hook is to have a single folder that contains all your HDL sources (subfolders are allowed). In most of my HDL projects, that folder is _sources/hdl._ I also configure my _.gitignore_ file to ignore all other sources in that folder, like this:

```shell
!sources/hdl/*.vhd
!sources/hdl/**/*.vhd
```

My pre-commit hook is below:

```sh
#!/bin/sh
hdl_folder="sources/hdl"

# Check if there are hdl files which are not commited
untracked_hdl_files=`git ls-files -o --exclude-standard ${hdl_folder}`

if ! [ -z $untracked_hdl_files ]; then
	echo "There are uncommited vhd files in ${hdl_folder}! Remove or add then!"
	echo "Files uncommited:"
	echo "$untracked_hdl_files"
	exit 1
fi

# If Emacs is available, this command will apply:
# - delete all trailing whitespace
# - run vhdl-beautify-buffer to all hdl sources
# - replace multiple empty lines with a single one

if command -v emacs &> /dev/null; then
# Check only hdl files that have changed
hdl_files=`git diff --name-only HEAD ${hdl_folder}`
for i in $hdl_files; do
	emacs -batch $i --eval '(progn (delete-trailing-whitespace) (vhdl-beautify-buffer) (replace-regexp "^\n+" "\n"))' -f save-buffer \
		# send all messages to trash
		2> /dev/null
done
# Add modified files
git add $hdl_files
fi
```

If you find this useful, be sure to let me know by emailing me! I also welcome any feedback and/or suggestions!
