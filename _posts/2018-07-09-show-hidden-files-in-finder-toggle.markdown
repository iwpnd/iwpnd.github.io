---
layout: post
title:  "Show hidden files in finder terminal alias"
date:   2018-07-09 20:27:55 +0200
categories: macos zsh terminal
---

MacOS has the annoying perk to keep you on training wheels unless you show your worth. So, in order to keep my mum and grandma from deleting important files, MacOS hides them by default. To show them, you can either always use the shortcut `Command + Shift + Period` whenever you want to toggle hidden files. Or you can add an alias to `zsh`.

To do that you open `.zshrc` that you find in your root directory and add the following to your terminal.

{% highlight python %}
alias show_hidden='defaults write com.apple.Finder AppleShowAllFiles YES && killall Finder'
alias hide_hidden='defaults write com.apple.Finder AppleShowAllFiles NO && killall Finder'
{% endhighlight %}

This lets you trigger to show/hide hidden files with a more permanent effect.

While you're at it, do yourself a favor and add another alias:

{% highlight python %}
alias zshconfig="vim ~/.zshrc"
{% endhighlight %}

That'll take you to the zsh config quickly in the future.
