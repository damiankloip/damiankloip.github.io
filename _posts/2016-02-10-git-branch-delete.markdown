---
layout: post
title: Delete Git Branches By Prefix
date: 2016-02-10
tags:
  - git
  - ruby
---

Here is a quick script I use to delete local git branches by a prefix. I like to
branch a lot, so I usually end up with tons of branches related to tickets etc..
This provides a simple way to remove them by a particular pattern. Yes, you could
just use an alias and one of the commands in the comment below. However, that does not
really give you the chance to easily see the branches that will be removed and confirm.
With a simple Ruby script, this can be done in a couple of minutes. Plus it's more fun!

<!--more-->

{% highlight ruby %}
#! /usr/bin/env ruby

# Ruby wrapper script to filter git branches by prefix, provide a confirmation
# prompt, then delete those branches. Without confirmation prompt you can easily
# do something like `git branch -D (git branch | egrep "^\s+core-" | tr -d ' ')`
# or `(git branch | egrep "^\s+core-") | xargs git branch -D`. This could be
# done with a bash script, but Ruby is pleasant to use.

prefix = ARGV.first
remote = ARGV[1]

raise 'No prefix specified' if prefix.nil?

branches = `git branch`
filtered = branches.split.select { |b| b =~ %r{^\s*#{prefix}} }

if filtered.empty?
  puts "No branches matching '#{prefix}'"
  exit
end

puts 'Matched branches:'
puts ''
puts filtered
puts ''

prompt = 'confirm? [y/n]: '.freeze

print prompt

# Create a loop that we break out of on 'y'/'n' only.
while input = STDIN.gets.chomp.downcase
  case input
  when 'y'
    # Just break, deal with branch deletion below.
    break
  when 'n'
    # Exit the script on 'n'.
    exit
  else
    # Was not 'y' or 'n', retry.
    print prompt
  end
end

puts ''

system('git', 'branch', '-D', *filtered)

# If origin branch has been specified. Remove these branches on the remote too.
# Note: This --delete option requires git >= 1.7.0.
if (remote)
  puts "Removing branches on remote: #{remote}"
  puts ''

  system('git', 'push', remote, '--delete', *filtered)
end
{% endhighlight %}