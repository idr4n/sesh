# A Sesh Fork

This is a [Sesh fork](https://github.com/joshmedeski/sesh) prior to v2.0.0.

It is likely that I will not be keeping this fork up to date with the original repo as it working for what I need in terms of managing Tmux sessions.

The additional feature in this fork (a PR that was not merged in Sesh), is to allow for an extra list of paths `path_list` in the session configuration file `$HOME/.config/sesh/sesh.toml`. In this way, when the session is first created, it will also create additional windows within the session for each path in `path_list`. 

For example:

```toml
[[session]]
name = "Things"
path = "~/Downloads"
startup_command = "ls"
path_list = ["~/Documents", "~/Dev"]
```

The main path is still the one defined in `path`, and the `startup_command` will only run in that window (i.e., in the `~/Downloads` window in the example above).

The rest of this README is based on [Sesh](https://github.com/joshmedeski/sesh) prior to v2.0.0.

## How to use

### tmux for sessions

[tmux](https://github.com/tmux/tmux) is a powerful terminal multiplexer that allows you to create and manage multiple terminal sessions. Sesh is designed to make managing tmux sessions easier.

### zoxide for directories

[zoxide](https://github.com/ajeetdsouza/zoxide) is a blazing fast alternative to `cd` that tracks your most used directories. Sesh uses zoxide to manage your projects. You'll have to set up zoxide first, but once you do, you can use it to quickly jump to your most used directories.

### Basic usage

Once tmux and zoxide are setup, `sesh list` will list all your tmux sessions and zoxide results, and `sesh connect {session}` will connect to a session (automatically creating it if it doesn't exist yet). It is best used by integrating it into your shell and tmux.

#### fzf

The easiest way to integrate sesh into your workflow is to use [fzf](https://github.com/junegunn/fzf). You can use it to select a session to connect to:

```sh
sesh connect $(sesh list | fzf)
```

#### tmux + fzf

In order to integrate with tmux, you can add a binding to your tmux config (`tmux.conf`). For example, the following will bind `ctrl-a T` to open a fzf prompt as a tmux popup (using `fzf-tmux`) and using different commands to list active sessions (`sesh list -t`), configured sessions (`sesh list -c`), zoxide directories (`sesh list -z`), and find directories (`fd...`).

```sh
bind-key "T" run-shell "sesh connect \"$(
	sesh list | fzf-tmux -p 55%,60% \
		--no-sort --ansi --border-label ' sesh ' --prompt '⚡  ' \
		--header '  ^a all ^t tmux ^g configs ^x zoxide ^d tmux kill ^f find' \
		--bind 'tab:down,btab:up' \
		--bind 'ctrl-a:change-prompt(⚡  )+reload(sesh list)' \
		--bind 'ctrl-t:change-prompt(🪟  )+reload(sesh list -t)' \
		--bind 'ctrl-g:change-prompt(⚙️  )+reload(sesh list -c)' \
		--bind 'ctrl-x:change-prompt(📁  )+reload(sesh list -z)' \
		--bind 'ctrl-f:change-prompt(🔎  )+reload(fd -H -d 2 -t d -E .Trash . ~)' \
		--bind 'ctrl-d:execute(tmux kill-session -t {})+change-prompt(⚡  )+reload(sesh list)'
)\""
```

You can customize this however you want, see `man fzf` for more info on the different options.

## gum + tmux

If you prefer to use [charmblacelet's gum](https://github.com/charmbracelet/gum) then you can use the following command to connect to a session:

```sh
bind-key "K" display-popup -E -w 40% "sesh connect \"$(
	sesh list -i | gum filter --limit 1 --placeholder 'Pick a sesh' --height 50 --prompt='⚡'
)\""
```

**Note:** There are less features available with gum compared to fzf, but I found its matching algorithm is faster and it ha a more modern feel.

See my video, [Top 4 Fuzzy CLIs](https://www.youtube.com/watch?v=T0O2qrOhauY) for more inspiration for tooling that can be integrated with sesh.

## zsh keybind

If you use zsh, you can add the following keybind to your `.zshrc` to connect to a session:

```sh
function sesh-sessions() {
  {
    exec </dev/tty
    exec <&1
    local session
    session=$(sesh list -t -c | fzf --height 40% --reverse --border-label ' sesh ' --border --prompt '⚡  ')
    [[ -z "$session" ]] && return
    sesh connect $session
  }
}

zle     -N             sesh-sessions
bindkey -M emacs '\es' sesh-sessions
bindkey -M vicmd '\es' sesh-sessions
bindkey -M viins '\es' sesh-sessions
```

After adding this to your `.zshrc`, you can press `Alt-s` to open a fzf prompt to connect to a session.

## Recommended tmux Settings

I recommend you add these settings to your `tmux.conf` to have a better experience with this plugin.

```sh
bind-key x kill-pane # skip "kill-pane 1? (y/n)" prompt
set -g detach-on-destroy off  # don't exit from tmux when closing a session
```

## Configuration

You can configure sesh by creating a `sesh.toml` file in your `$XDG_CONFIG_HOME/sesh` or `$HOME/.config/sesh` directory.

```sh
mkdir -p ~/.config/sesh && touch ~/.config/sesh/sesh.toml
```

### Default Session

The default session can be configured to run a command when connecting to a session. This is useful for running a dev server or starting a tmux plugin.

```toml
[default_session]
startup_command = "nvim -c ':Telescope find_files'"
```

You can also use the `startup_script` property to run a script when connecting to a session.

```toml
[default_session]
startup_script = "nvim -c ':Telescope find_files'"
```

**Note:** To learn how to write startup scripts, see the [startup script section](#startup-script).

### Session Configuration

A startup script is a script that is run when a session is created. It is useful for setting up your environment for a given project. For example, you may want to run `npm run dev` to automatically start a dev server.

**Note:** If you use the `--command/-c` flag, then the startup script will not be run.

I like to use a script that opens nvim on session startup:

```toml
[[session]]
name = "Downloads 📥"
path = "~/Downloads"
startup_command = "ls"

[[session]]
name = "tmux config"
path = "~/c/dotfiles/.config/tmux"
startup_command = "nvim tmux.conf"
```

### Listing Configurations

Session configurations will load by default if no flags are provided (the return after tmux sessions and before zoxide results). If you want to explicitly list them, you can use the `-c` flag.

```sh
sesh list -c
```

### Startup Script

A startup script is a simple shell script that is run when a session is created. It is useful for setting up your environment for a given project. For example, you may want to run `npm run dev` to automatically start a dev server and open neovim in a split pane.

```sh
#!/usr/bin/env bash
tmux split-window -v -p 30 "npm run dev"
tmux select-pane -t :.+
tmux send-keys "nvim" Enter
```

Set the file as an executable and it will be run when you connect to the specified session.

## Background (the "t" script)

Sesh is the successor to my popular [t-smart-tmux-session-manager](https://github.com/joshmedeski/t-smart-tmux-session-manager) tmux plugin. After a year of development and over 250 stars, it's clear that people enjoy the idea of a smart session manager. However, I've always felt that the tmux plugin was a bit of a hack. It's a bash script that runs in the background and parses the output of tmux commands. It works, but it's not ideal and isn't flexible enough to support other terminal multiplexers.

I've decided to start over and build a session manager from the ground up. This time, I'm using a language that's more suited for the task: Go. Go is a compiled language that's fast, statically typed, and has a great standard library. It's perfect for a project like this. I've also decided to make this session manager multiplexer agnostic. It will be able to work with any terminal multiplexer, including tmux, zellij, Wezterm, and more.

The first step is to build a CLI that can interact with tmux and be a drop-in replacement for my previous tmux plugin. Once that's complete, I'll extend it to support other terminal multiplexers.

## Contributors

<a href="https://github.com/joshmedeski/sesh/graphs/contributors">
  <img src="https://contrib.rocks/image?repo=joshmedeski/sesh" />
</a>

Made with [contrib.rocks](https://contrib.rocks).
