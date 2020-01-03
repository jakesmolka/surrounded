---
title: Time Tracking and System Integration via Zsh
#author: Jake
date: '2020-01-03'
categories:
  - Laptop
tags:
  - Arch
  - Applications
slug: time-tracking-system-integration-zsh
---

# Time Tracking and System Integration via Zsh

As part of my first day at work this year I thought cleaning up my setup would be a good idea. The first thing I do most days is starting my time tracking tool. Until today I was using some web service that was available some years ago when I was looking for one, somewhat glossy and backed up on some server. Especially that last point was important for me at that time.

But especially the last point turned out to be not important anymore. Luckily, I don't gather tons of overtime (or negative time) which would be very important to track over a long time. So, the requirement for some kind of web server or direct syncing isn't up to date for my time tracking anymore - a local application would be sufficient. Additional, I'm backing up my working laptop hourly anyway.

## Application

My first choice when looking for a new application is the Arch Wiki. But the [listed time trackers](https://wiki.archlinux.org/index.php/List_of_applications#Time_trackers) weren't really what I was looking for. Seems like an older list, which also includes entries from a variety of timers in general (e.g. stop watches etc.).

Then I found this [linuxlinks.com list](https://www.linuxlinks.com/timetrackers/) of CLI time trackers. Alternatively, the Arch Wiki and linuxlinks.com also lists several GUI trackers, but they are often out dated or bloated in my opinion.

The list consists of five applications:

- **Watson**: Important advantage: Was recommended to me by a friend. Very active project, very good documentation website. Good set of features, that is not over-complicating things, but allows *projects*, *tags*, *reports*, export and so on. See the [overview here](https://tailordev.github.io/Watson/).
- **timetrap**: Rather inactive project in comparison. But functionality seems alright. Natural Language Times is a nice feature.
- **doing**: Is more than time tracking and seems to be build around a file format of some Mac app I can't use. Other goal than my use-case and too complex if only used for time tracking.
- **ctt**: Rather inactive project in comparison. Also has no welcoming README or documentation explaining the basic functionality. Was reading the manpage-like file in repo, but I don't like nor understand why some projects are kept unwelcoming like this.
- **utt**: Basic functionality appears to be like Watson, but somewhat reverted workflow: Start workday and add tasks to the report when done. Doesn't match the workflow I was expecting and has no advantages to make me change to it.

### Watson

I chose Watson. Despite the advantages and pros it also has some cons or missing features (from my perspective, of course):

- Time needs to be entered in specific format, Natural Language Times would be helpful. But there is [work going on in this direction](https://github.com/TailorDev/Watson/issues/278).
- Starting a tracking process starting at a time in the past, like `watston start (options) --from $TIME` isn't possible. It is possible to directly start where the former frame was stopped with `watson start --no-gap`, but that won't allow me to, say, start working at 9am but remember to turn tracking on at 10am, when the last frame was from yesterday evening. What's left is to `watson start`, `watson stop` and `waton edit -1` editing the start time manually followed by `watson restart`. Rather annoying, right?

Two requirements were perceived as missing first, but turned out to be available:

- (Found the solution while writing, so I'm preserving the problem.) ~~I couldn't manage to get a view / report showing me the aggregated tracked time of the day regardless of amount of sub-times (frames) or state (already stopped frame or still running frame), i.e. my current work time of the current day. Currently I would need to stop the running tracking `watson stop`, call `watson report --day` and restart `watson restart` OR add the numbers given by `watson report --day` and `watson status`. Obviously, this is way too complicated for a simple problem / requirement like that.~~ `watson report -dc` does exactly what I was looking for. It combines `--day` and `--current`. There is also a config entry allowing to include the *current* frame into the *report* by default.
- Stopping the current tracking at a given time, when forgetting to stop it in time, works with `watson stop --at $TIME`.

## Integration

With my former solution I had an always open browser tab with visual indication wether or not the tracking was running. Basically, this helped me to remember to turn tracking on and off. Especially with an CLI tool, I was sure I'd need some kind of indicator for *Watson* as well.

Some web searching later I found a little [bash config modification](https://tailordev.fr/blog/2016/03/31/watson-community-tools/) which adds a little green or red symbol in front of the bash prompt when *Watson* is running or stopped, respectively. 

Inspired by that I tried to make this work in *Zsh*, my shell of choice. Before that I never really tried to modify my *Zsh* and was glad there's a nice existing config easily available (I liked the **grml** one the Arch install ISO uses, see [here](https://wiki.archlinux.org/index.php/Zsh#Sample_.zshrc_files)). So I tried this and that, read examples and man pages here and there, but couldn't figure out how to alter my prompt. Annoyed and demotivated to continue that path I decided to just escalate that problem by using *Zsh* plugins or a framework. 

While researching for something that might help me I found the [Zsh Spaceship prompt](https://github.com/denysdovhan/spaceship-prompt), which technically is a theme I think, but offers an own API for extension. To get it into my system I first tried to use the [Antibody](https://github.com/getantibody/antibody) *Zsh* plugin manager, but couldn't get my *Zsh* to recognize *Spaceship* as available prompt style/theme. Luckily the Arch AUR has a `spaceship-prompt-git` package (and an official `otf-fira-code` package for a font required by *Spaceship*), which is to favor anyway to keep updating and general package handling in one hand: my system's package manager.

To activate *Spaceship* the last line from this snippet was added to my `.zshrc`. The first three lines are probably present in almost any existing config (according to [Arch Wiki's Simple .zshrc](https://wiki.archlinux.org/index.php/Zsh#Simple_.zshrc)).

```
autoload -Uz compinit promptinit
compinit
promptinit
prompt spaceship # via AUR package 'spaceship-prompt-git'
```

(Don't forget to `source ~/.zshrc` to reload it.)

At this point I had time tracking via *Watson* and a fancy *Zsh* prompt via *Spaceship*, but both parts weren't integrated at all, yet. To achieve that I was going to replicate the *bash* example from above utilizing the *Spaceship* API, at least that was my plan. While reading about *Spaceship's* feature to add custom sections I luckily found this gem: a custom *Spaceship* section named [Watson current working project](https://github.com/denysdovhan/spaceship-prompt/wiki/External-sections#watson-current-working-project) (please see [the linked actual source](https://github.com/letientai299/dotfiles/blob/master/spaceship/watson.zsh) for newer version). It checks *Watson* for running tracking and display the name of the project if *Watson* is currently tracking.

So I created a `~/.zsh_spaceship_watson.zsh` file with a copy of the code above, but little modifications to add an indication even if *Watson* is not running, see:

``` bash
#!/bin/zsh

# ------------------------------------------------------------------------------
# Configuration
# ------------------------------------------------------------------------------

SPACESHIP_WATSON_SHOW="${SPACESHIP_WATSON_SHOW=true}"
SPACESHIP_WATSON_SYMBOL="${SPACESHIP_WATSON_SYMBOL="🔨 "}"
SPACESHIP_WATSON_PREFIX="${SPACESHIP_WATSON_PREFIX="$SPACESHIP_PROMPT_DEFAULT_PREFIX"}"
SPACESHIP_WATSON_SUFFIX="${SPACESHIP_WATSON_SUFFIX="$SPACESHIP_PROMPT_DEFAULT_SUFFIX"}"
SPACESHIP_WATSON_COLOR="${SPACESHIP_WATSON_COLOR="green"}"

# ------------------------------------------------------------------------------
# Section
# ------------------------------------------------------------------------------

# Show watson status
# spaceship_ prefix before section's name is required!
# Otherwise this section won't be loaded.
spaceship_watson() {
  # If SPACESHIP_WATSON_SHOW is false, don't show watson section
  [[ $SPACESHIP_WATSON_SHOW == false ]] && return

  # Check if watson command is available for execution
  spaceship::exists watson || return

  # Use quotes around unassigned local variables to prevent
  # getting replaced by global aliases
  # http://zsh.sourceforge.net/Doc/Release/Shell-Grammar.html#Aliasing
  local 'watson_status'
  watson_status=$(watson status | perl -pe 's/(Project | started.*\(.*\))//g')

  # Exit section if variable is empty
  [[ -z $watson_status ]] && return
  # For default "watson is not running" response create red "OFF" section content
  [[ $watson_status =~ 'No project started' ]] && watson_status="OFF" && SPACESHIP_WATSON_COLOR="red" #return

  # Display watson section
  spaceship::section \
    "$SPACESHIP_WATSON_COLOR" \
    "$SPACESHIP_WATSON_PREFIX" \
    "$SPACESHIP_WATSON_SYMBOL$watson_status" \
    "$SPACESHIP_WATSON_SUFFIX"

}
```

In the last step I had to activate this custom section in my `~/.zshrc`, see (also includes my general *Zsh* settings):

``` bash
# Lines configured by zsh-newuser-install
HISTFILE=~/.histfile
HISTSIZE=1000
SAVEHIST=1000
bindkey -e
# End of lines configured by zsh-newuser-install
# The following lines were added by compinstall
zstyle :compinstall filename '~/.zshrc'

autoload -Uz compinit promptinit
compinit
promptinit
prompt spaceship # via AUR package 'spaceship-prompt-git'
# End of lines added by compinstall

# spaceship section config. comment out to disable.
SPACESHIP_PROMPT_ORDER=(
  time          # Time stamps section
  user          # Username section
  dir           # Current directory section
  host          # Hostname section
  git           # Git section (git_branch + git_status)
  watson
  #hg            # Mercurial section (hg_branch  + hg_status)
  package       # Package version
  node          # Node.js section
  #ruby          # Ruby section
  #elixir        # Elixir section
  #xcode         # Xcode section
  #swift         # Swift section
  golang        # Go section
  #php           # PHP section
  #rust          # Rust section
  #haskell       # Haskell Stack section
  #julia         # Julia section
  #docker        # Docker section
  #aws           # Amazon Web Services section
  venv          # virtualenv section
  #conda         # conda virtualenv section
  #pyenv         # Pyenv section
  #dotnet        # .NET section
  #ember         # Ember.js section
  kubecontext   # Kubectl context section
  terraform     # Terraform workspace section
  exec_time     # Execution time
  line_sep      # Line break
  battery       # Battery level and status
  vi_mode       # Vi-mode indicator
  jobs          # Background jobs indicator
  exit_code     # Exit code section
  char          # Prompt character
)
# time section custom options
SPACESHIP_TIME_SHOW=true
# dir section custom options
SPACESHIP_DIR_TRUNC_PREFIX=…/
# custon watson section loaded (already in order above)
source ~/.zsh_spaceship_watson.zsh

# manual loading module - files installed via 'zsh-syntax-highlighting' package
source /usr/share/zsh/plugins/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh

# golang
export GOPATH="/home/jake/go"
export PATH="$PATH:$GOPATH/bin"
```

Disadvantages / Cons:

- Spaceship makes the prompt way slower. Time will show if I think the given features are worth it or not. Alternatively, I might invest some time in fine tuning the config and maybe disable more features to speed it up.
- Additionally, I need to adjust to the changes. I was using my prompt style for some time now and probably will be confused now and then until I finally accept the new order, colors and style in general.
- Why didn't it work to alter my prompt like in the bash example? Really dissatisfying! I hope I'll be motivated enough some day to figure it out and solve this mystery.

Altogether my prompt and its *Watson* integration looks like this. First tracking is off, then it's running and stopped again:

![](blog-watson-better.png)