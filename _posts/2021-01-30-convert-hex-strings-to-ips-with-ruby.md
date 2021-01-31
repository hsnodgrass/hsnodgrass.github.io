---
title: Converting Hex Strings to IP Addresses with Ruby
author: Heston Snodgrass
date: 2021-01-30 17:00:00 -0700
catagories: [Ruby]
tags: [ipv4, linux, socket, hexidecimal, ruby]
math: true
mermaid: true
---

This past week I found myself wanting to parse `/proc/net/tcp` using pure Ruby. Specifically, I
wanted to convert parts of the socket information, specifically the IP and port, to a
human-readable form. IP addresses and ports are represented in `/proc/net/tcp` as hexidecimal
strings and I had no idea how to decode them to be human readable.

### TL;DR

```ruby
# Decode a hex string representation of an IPv4 address
["<hex IP string>".to_i(16)].pack('L').unpack('CCCC').join('.')

# Decode a hex string of a TCP / UDP port
"<hex port string>".to_i(16)
```

## The explanation

As with all issues that stump me while coding, I immediately googled it. I found a brief [post on
StackOverflow](https://stackoverflow.com/questions/43855030/how-to-manually-convert-an-integer-into-an-ip-address-in-ruby)
that had some code that worked for converting integers to IP addresses, but not the hex to integers.
Since there was a gap between what I wanted to do and what that code does, I had to figure out
how to convert a hex string into an integer, then figure out why that code worked.

[IPv4 addresses]((https://en.wikipedia.org/wiki/IPv4)) are really just 32-bit integers. So, the first
thing that needs to be done when decoding the hexidecimal representation of the IP is to convert
it to it's native form. In Ruby, this is silly easy because the `.to_i` method allows you to pass
a [radix](https://en.wikipedia.org/wiki/Radix) as a parameter. Since hexidecimal is *base 16*,
we can get a numeral representation of hexidecimal string like so:

```ruby
'bc4ff2'.to_i(16) # => 12341234

12341234.to_s(16) # => "bc4ff2"
```

I forgot to mention, just like `to_i` allows us to pass a radix, `.to_s` also does. This makes
converting between integers and hex really easy. So now we can decode our port from the socket,
however getting a human-readable IP address takes a few more steps.

For the next part, I needed to use two `Array` methods I'd never used before: `.pack` and `.unpack`.
What these methods do is convert the contents of the array to a binary string using a template, and
vise-versa. Why do we need to do this? Because it's easier than writing a function to perform the
conversion ourselves, of course!

So, since IPv4 addresses are just 32-bit unsigned integers, we get a binary representation of one
like so:

```ruby
# Lets assume "C0A80001" is the hex-form IP we get from /proc/net/tcp
ip_int = "C0A80001".to_i(16) # => 3232235521
ip_bin = [ip_int].pack('N') # => "\x01\x00\xA8\xC0"
```

Wait, what does the `N` mean? That's the template we're using to convert the integer to binary. The
`N` means we are converting a 32-bit unsigned, [big-endian](https://en.wikipedia.org/wiki/Endianness) integer. Also, notice how there are
four sections in the binary string. Matches up nicely with the four *octets*, or 8-bit sequences,
that make up an IPv4 address.

> **NOTE** We should technically use `N` as the template because that means 32-bit unsigned,
> big-endian integer which is, according to my layman's knowlege, what IPv4 addresses are
> supposed to use. However, on my CentOS 7 VM, using `N` as the template rendered the IP
> addresses backwards when decoding them, so I used `L`, which is 32-bit unsigned, native-endian
> integer and made the operating system decide, which worked.

Now we need to get that binary string into a more palatable form. Fortunately, the `C` template string
means 8-bit unsigned integer which is exactly what we need for the octets in the IP:

```ruby
ip_octets = ip_bin.unpack('CCCC') # => [192, 168, 0, 1]
ip_addr = ip_octets.join('.') # => "192.168.0.1"

# Putting it all together
ip_addr = ["C0A80001".to_i(16)].pack('N').unpack('CCCC').join('.') # => "192.168.0.1"
```

*While writing this post, I found [this converter](https://www.vultr.com/resources/ipv4-converter/) for the examples which makes a good bookmark.*
