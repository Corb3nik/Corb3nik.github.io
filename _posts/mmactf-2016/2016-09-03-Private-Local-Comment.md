---
layout: post
title: Private / Local / Comment
category: MMA-CTF-2016
---

## Description
This is a three part challenge testing our Ruby knowledge.


## The Challenge

This challenge was split into three parts, each one harder than the last. The goal
was to access a flag that isn't normally accessible through basic object oriented functionalities.

## Private

For each challenge, the source code was given. Here is the source code for the `Private` challenge :

``` ruby
require_relative 'restrict'
Restrict.set_timeout

class Private
  private
  public_methods.each do |method|
    eval "def #{method.to_s};end"
  end

  def flag
    return "TWCTF{CENSORED}"
  end
end

p = Private.new
Private = nil

input = STDIN.gets
fail unless input
input.size > 24 && input = input[0, 24]

Restrict.seccomp

STDOUT.puts eval(input)
```

The source code above redefines all `public` functions as `private` for the `Private` class, making it difficult for us to call the `flag` function.

The solution here is to [define a method for a single instance](http://stackoverflow.com/questions/803020/redefining-a-single-ruby-method-on-a-single-instance-with-a-lambda).

We did have a limit of 24 characters that can be used as our final payload, so
here is the final payload :

  ![private](/assets/img/mma-ctf-2016/Private.png "private")

#### Local

Here is the source code for the `Local` challenge :

```ruby
require_relative 'restrict'
Restrict.set_timeout

def get_flag(x)
  flag = "TWCTF{CENSORED}"
  x
end

input = STDIN.gets
fail unless input
input.size > 60 && input = input[0, 60]

Restrict.seccomp

STDOUT.puts get_flag(eval(input))
```

In this challenge, we had to figure out a way to read a local `flag` variable.

Since the flag is loaded in memory, I've decided to use the `ObjectSpace` class to iterate through the loaded strings in memory.

The challenge gave us a bit more space for our final payload (60 characters).

Here is the final payload : `ObjectSpace.each_object(String){|x|p x if (/TF.{8,}}$/ =~ x)}`

  ![local](/assets/img/mma-ctf-2016/Local.png "local")

#### Comment

Doing this challenge, I noticed that my solution from the previous was probably not the intended one.

Here is the source code for the `Comment` challenge :

```ruby
# comment.rb
require_relative 'restrict'
Restrict.set_timeout

input = STDIN.gets
fail unless input
input.size > 60 && input = input[0, 60]

require_relative 'comment_flag'
Restrict.seccomp

STDOUT.puts eval(input)
```

```ruby
# comment_flag.rb
# FLAG is TWCTF{CENSORED}
```

This challenge consisted of finding a commented flag that is loaded into the main script through `require_relative`.

We can use the same code from the previous challenge to obtain the flag :

  ![comment](/assets/img/mma-ctf-2016/Comment.png "comment")

