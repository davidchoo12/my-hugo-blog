---
title: "List of linux commands"
slug: list-of-linux-commands
tags: [linux]
published: 2021-06-23
---

Here is a list of linux cli commands/tools that I have used. Purpose is for my own future reference and to track what I know. The list is non exhaustive and not sorted in any way. [explainshell.com](https://explainshell.com) is a very nice website to learn about the commands, can bookmark `https://explainshell.com/explain?cmd=%s`.

## Shell tools
- `man`: get help for commands
- `which`: get first path of command
- `where`: get paths of command
- `alias`: set command shortcut
- `whoami`: get current user
- `watch`: runs command every n seconds
  - `watch -n5`: runs command every 5 sec
- `tee`: split stdin to file and stdout
- `xargs`: reads stdin line by line and run command for every line
  - `xargs -I{} <command> "{}"`: set each line as {} and run command with the line as first arg
- `set`: sets shell options, usually used at the top of a bash script
  - `set -euo pipefail`: bash strict mode
  - `set -euxo pipefail`: bash strict mode, also prints each line
- `test`: check condition and set exit code 0 for true, 1 for false, usually used in if statements
  - `[`: alias of test, for writing bash script like if [ condition ]
  - `[[`: better version of [

## Text operations
- `echo`: prints text
- `cat`: prints file content, also used for con"cat"enating files
- `less`: reads file in a scrollable interface (ncurses), use j and k to scroll per line or u and d to scroll per page, q to quit
  - `more`: older version of less, cannot scroll up
- `head`: reads first n lines
  - `head -n5`: reads first 5 lines
- `tail`: reads last n lines
- `grep`: find a word or with regex in files
- `sed`: search and replace
- `tr`: simpler alternative to sed
- `cut`: split string and select substring
  - `cut -d' ' -f1`: split by space, select first word
- `awk`: text manipulation tool, includes its own scripting language (heard of it, never used it)
- `sort`: sort lines
- `uniq`: deduplicate adjacent lines, usually used after sort
- `wc`: word count
  - `wc -l`: line count
  - `sort <file> | uniq | wc -l`: count unique lines
- `jq`: reads json and prints selected key, includes its own scripting language

## File operations
- `ls`: prints directory listing
  - `ls -lah`: prints in detail
  - `ls -lahtr`: sorted by last modified (newest first)
- `cd`: change directory
- `z`: oh-my-zsh plugin to change to recent directory
- `rm`: remove file/directory
  - `rm -rf`: removes recursively
- `cp`: copies file
- `mv`: moves file
- `mkdir`: creates directory
- `pwd`: prints working(current) directory, not to be confused with passwd
- `tar`: compress or extract tar files
  - `tar -xzvf`: extracts .tar.gz files
- `gzip`: compress or extract gzip files
  - `gzip -d`: extracts .gz file in the same dir, removing the .gz file
  - `gzip -dk`: extracts .gz file, keeping the .gz file
- `zless`: `less` for .gz file without extracting it
- `zcat`: `cat` for .gz file without extracting it
- `zgrep`: `grep` for .gz file without extracting it
- `zip`: compress to .zip
- `unzip`: extracts .zip
- `touch`: creates file, sets last modified time
- `chmod`: change file permissions
  - `chmod +x`: make file executable
- `ln`: creates symbolic link
- `find`: find a file by filename

## System tools
- `uname`: prints OS name
  - `uname -a`: prints all OS info
- `env`: prints all env vars
  #!/usr/bin/env bash: runs bash specified in $PATH, usually used in first line of bash script
- `export`: sets user env var
- `reboot`: restarts machine
- `chroot`: treat directory as fake root dir, containers are built upon this (heard of it, never used it)
- `lshw`: lists all hardware info
- `lscpu`: prints cpu info
- `df`: show disk utilization

## Process tools
- `ps`: list running processes
  - `ps aux`: list all
  - `ps ef`: alternative to ps aux
- `htop`: like task manager in windows, view system resources
- `top`: older colourless version of htop
- `dstat`: prints system resource utilization every second
- `kill`: kills process by sending SIGTERM
  - `kill -9`: kills using SIGKILL (cannot be caught by process)
  - `kill -s`: kills using specified signal

## Network tools
- `netstat`: list open ports
  - `netstat nltp`: list all TCP listening ports and its process, may need sudo
- `ss`: alternative to netstat
- `ifconfig`: like ipconfig in windows, shows local IP address
- `ip`: alternative to ifconfig
  - `ip a`: show ip address
- `ping`: like ping in windows, sends ICMP packet to check if address is accessible and measure round trip time
- `tracert`: measure hop by hop round trip time
- `dig`: like nslookup in windows, runs dns lookup
- `ssh`: provides a shell to access remote machine
- `scp`: file transfer over ssh
- `rsync`: more optimized alternative to scp
- `pssh`: sends ssh commands in parallel to multiple machines
- `i2cssh`: only for iterm2, opens multiple iterm2 panels to ssh into multiple machines (feels very hackery lol)
- `who`: see who else is ssh-ing into the same machine
- `curl`: send http request, also includes various other protocols
  - `curl -fsSL`: send request non verbose, follows http redirects
  - `curl -X POST -d 'param1=value1&param2=value2'`: posts x-www-form-urlencoded
  - `curl -X POST -H 'content-type: application/json' -d '{"a":"b"}'`: posts json
  - `curl -F 'image=@file.jpg' -F 'param=value'`: posts multipart/form-data

## User permission
- `sudo`: runs as root
  - `sudo -s`: change as root user, use my default shell, start in same directory
- `passwd`: change password

## Other tools
- `bc`: basic calculator
- `date`: prints current datetime

## Shells
- `zsh`: Z shell, popular cos of oh-my-zsh, default shell of MacOS, Arch
- `bash`: Bourne again shell, popular cos default shell of Ubuntu
- `sh`: symbolic link to default shell
- `fish`: fish shell
- `chsh`: change default shell

## Text editors
- `vim`: very customizeable, not too beginner friendly text editor
- `vi`: older alternative to vim
- `nvim`: neovim, newer alternative to vim
- `nano`: beginner friendly text editor

## Git
- `git`: version control system, best to alias all the commands using oh-my-zsh git plugin
  - `git status`: show status
  - `git log`: show commit history
    - `git log --all --decorate --oneline --graph`: pretty log
  - `git add`: stages file
  - `git commit`: commits staged changes
    - `git commit -m`: 
    - `git commit --amend --no-edit`: rewrite previous commit with staged changes
  - `git push`: pushes commits
    - `git push -f`: force overwrite remote commits
    - `git push --force-with-lease`: force if no newer commits on remote
  - `git pull`: pulls commits
  - `git reset`: unstage changes
    - `git reset HEAD~`: undo previous commit
    - `git reset --hard HEAD~`: undo previous commit and undo changes in working directory
  - `git checkout`: change branch
    - `git checkout -b`: create new branch and change branch
  - `git merge`: merge branches
  - `git remote`: show remote name
    - `git remote -v`: list remote repos
  - `git rebase`: moves branch's base commit up to latest by rewriting commits, good for keeping clean history, bad if you're working in a team
    - `git rebase -i`: interactive rebase
    - `git rebase --abort`: cancel ongoing rebase
  - `git reflog`: list history of commits accessed in local repo, useful if you lost reference due to rebase or if you mess up rebasing

## Docker
- `docker`: for docker containers
  - `docker pull`: downloads image
  - `docker run`: run container
    - `docker run -it <container> /bin/bash`: enter the container (like ssh into container)
  - `docker images`: list images
  - `docker ls`: list containers
  - `docker login`: saves credential to remote docker registry (for pulling/pushing images)

## Package managers
- `apt`: for ubuntu
- `brew`: for mac
- `npm`: for nodejs
- `pip`: for python
  - `python -m venv .venv`: create virtual env
  . .venv/bin/activate: enter virtual env
  - `virtualenv -p <path to python2> .venv`: create virtual env for python2
- `cargo`: for rust
- `go get`: for golang
- `gem`: for ruby
- `fpm`: effing package management, creates various packages

## Version managers
- `pyenv`: for python
- `rbenv`: for ruby
- `rvm`: alternative to rbenv
- `nvm`: for nodejs

## Fancy tools
- `ranger`: file explorer in terminal
- `ncmpcpp`: mpd client for playing music in terminal