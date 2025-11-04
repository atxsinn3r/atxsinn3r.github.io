# A Performance Question: Regular Expression as a Security Fix

## Introduction

When writing a report for a vulnerability finding, one of the most common and generic things I say about fixing anything is "check user inputs". Whether this was a trained thing, or out of habit, sometimes my elabration woud further suggest adding a regular expression check. For example, if there is a ping feature that is vulnerable to a command injection attack due to the IP address being user-controlled, then a regex check for the IPv4 pattern would suffice.

Regular expression as a security check is great because every popular programming language supports it, and every developer knows how to use it. However, while the check would probably do the trick for the sake of security, looking back I'm not sure if it was always appropriate.

The thing about regular expression is that people, including me, are probably overusing it. We do so without considering the potential performance penality, and I'm sure many of us don't always write code with that in mind. As a security researcher, the truth is we don't always know the codebase like the developers do. I wonder if we are always putting ourselves in the developer's shoes, or we are asking them to trade performance over security without realizing that.

So that got me curious about this question: Assuming regular expression provides enough security, how much performance am I asking the developers to sacrifice, and is that worth it?

In other words, I am just curious how often my clients were possibly rolling their eyes behind the screen when my colleagues and I mentioned something they knew they couldn't implement, but too polite to call us out?

## Rules of the Experiment

To explore this topic, I wanted to get a feel of what fast is supposed to look like in different implementations, so I wrote a few IPv4 checks in C. And then I wrote them in Ruby, because generally people believe the language itself is just slower even with JIT.

I defined some rules about the check. On a high level, the check should:

* Tell me if an input is a legal IPv4 string, where each octet must be a decimal between 0 to 255.
* IPv4 in decimal format is not considered.
* The check is executed 0x8000 times.

For example, the check should accept this:

```
192.168.1.123
```

But will not accept this:

```
3232235899
```

## The Test Cases

The following table is a collection of cases I wrote for this experiment:

1. C with multiple regex_t compilations
2. C with one regex_t compilation
3. C without any regular expression

### Case #1: C with multiple regex_t compilations

I wanted to write this check because in my experience of code reading, this seems to be a rather common way to perform a regular expression check in C. The regcomp function call is placed inside some API and services multiple requests.

```c
#define IPV4_PATTERN "^([[:digit:]]+)\\.([[:digit:]]+)\\.([[:digit:]]+)\\.([[:digit:]]+)$"

bool IsIPv4(const char *val) {
  regex_t regx;
  regmatch_t matches[5];
  if (regcomp(&regx, IPV4_PATTERN, REG_EXTENDED) != 0) {
    return false;
  }

  char* part = NULL;
  bool result = false;
  if (regexec(&regx, val, 5, matches, 0) == 0) {
    for (int i=1; i < 5; i++) {
      char *start = (char *)val + matches[i].rm_so;
      size_t len = matches[i].rm_eo - matches[i].rm_so;
      char* part = strndup(start, len);

      // This checks:
      // 1) Makes sure strndup is not out of memory
      // 2) Makes sure each part is between 1 to 3 bytes
      if (!part || (!(strlen(part) > 0 && strlen(part) < 4))) {
        goto done;
      }

      // All bytes should be numeric
      char* refPart = part;
      while (*refPart) {
        if (!isdigit(*refPart)) {
          goto done;
        }
        refPart++;
      }

      // Each numeric value should be no larger than 255
      char* e = NULL;
      uint32_t partN = strtol(part, &e, 10);
      if ((e == part) || partN > 255) {
        goto done;
      }
      free(part);
    }
    result = true;
  }

done:
  free(part);
  regfree(&regx);
  return result;
}

int main(int args, char** argv) {
  const char *ip = "192.128.6.1";
  for (int i=0; i < 0x8000; i++) {
    printf("IsIPv4: %d\n", IsIPv4(ip));
  }
  return 0;
}
```

### Case #2: C with one regex_t compilation

My reasoning for this version is that I personally think having the same regex_t compiled repeatedly is redundant; it should only be compiled once whenever possible. This is my preferred version, even though I don't necessarily think it is always the best way for every programming scenario:

```c
#define IPV4_PATTERN "^([[:digit:]]+)\\.([[:digit:]]+)\\.([[:digit:]]+)\\.([[:digit:]]+)$"

bool IsIPv4(regex_t* regx, const char* val) {
  regmatch_t matches[5];
  char *part = NULL;
  bool result = false;
  if (regexec(regx, val, 5, matches, 0) == 0)
  {
    for (int i = 1; i < 5; i++)
    {
      char *start = (char *)val + matches[i].rm_so;
      size_t len = matches[i].rm_eo - matches[i].rm_so;
      char *part = strndup(start, len);

      // This checks:
      // 1) Makes sure strndup is not out of memory
      // 2) Makes sure each part is between 1 to 3 bytes
      if (!part || (!(strlen(part) > 0 && strlen(part) < 4)))
      {
        goto done;
      }

      // All bytes should be numeric
      char *refPart = part;
      while (*refPart)
      {
        if (!isdigit(*refPart))
        {
          goto done;
        }
        refPart++;
      }

      // Each numeric value should be no larger than 255
      char *e = NULL;
      uint32_t partN = strtol(part, &e, 10);
      if ((e == part) || partN > 255)
      {
        goto done;
      }
      free(part);
    }
    result = true;
  }

done:
  free(part);
  return result;
}

int main(int args, char** argv) {
  const char *ip = "192.128.6.1";
  regex_t rex;
  if (regcomp(&rex, IPV4_PATTERN, REG_EXTENDED) != 0) {
    return 0;
  }

  for (int i=0; i < 0x8000; i++) {
    printf("IsIPv4: %d\n", IsIPv4(&rex, ip));
  }
  regfree(&rex);
  return 0;
}
```

### Case #3: C with any regular expression

Obviously there needs to be a villian of this experiment to see if regular expression is worthy:

```c
bool IsIPv4(const char* val) {
  // At a minimum, the IPv4 string length should be no less than 0.0.0.0
  if (strlen(val) < strlen("0.0.0.0")) {
    return false;
  }

  char* ref = (char*) val;
  int parts = 0;
  while (ref && *ref) {
    char* delimiter = strchr(ref, '.');
    char* part = delimiter ? strndup(ref, (size_t) (delimiter - ref)) : strdup(ref);
    parts++;
    // This checks: 1) strdup is not out of memory; 2) there should be 4 parts
    if (!part || parts > 4) {
      return false;
    }

    // Part length should between 1 to 3 bytes.
    if (!(strlen(part) > 0 && strlen(part) < 4)) {
      return false;
    }

    // All bytes should be numeric.
    char* refPart = part;
    while (*refPart) {
      if (!isdigit(*refPart)) {
        return false;
      }
      refPart++;
    }

    // Each numeric value should no be no larger than 255
    char* end = NULL;
    uint32_t partN = strtol(part, &end, 10);
    if ((end == part) || partN > 255) {
      return false;
    }
    free(part);
    ref = delimiter ? delimiter + 1 : NULL;
  }
  return true;
}

int main(int args, char** argv) {
  const char *ip = "192.128.6.1";
  for (int i=0; i < 0x8000; i++) {
    printf("IsIPv4: %d\n", IsIPv4(ip));
  }
  return 0;
}
```

### Case #4: Ruby Multiple pattern compilations

In Ruby, I'd say #scan is my favorite method to capture groups in regular expression:

```ruby
#!/usr/bin/env ruby

def is_ipv4(val)
  groups = val.scan(/^(\d+{1,3})\.(\d+{1,3})\.(\d+{1,3})\.(\d+{1,3})$/)
  groups.each do |part|
    partN = part.flatten.first.to_i
    unless partN > 0 && partN < 255
      return false
    end
  end
  true
end

ip = '192.128.6.1'
(0..0x8000).each do |i|
  puts is_ipv4(ip)
end
```

### Case #5: Ruby with Manual Control of Compilation

The problem with the #scan method is that I may not have control over the compilation, so in theory this may be a better choice:

```ruby
def is_ipv4(r, val)
  m = r.match(val).to_a
  return false if m.empty?
  m.shift

  m.each do |part|
    partN = part.to_i
    unless partN > 0 && partN < 255
      return false
    end
  end

  true
end

ip = '192.128.6.1'
r = Regexp.new(/^(\d+{1,3})\.(\d+{1,3})\.(\d+{1,3})\.(\d+{1,3})$/)
0x8000.times.each do |i|
  puts is_ipv4(r, ip)
end
```

## Results

The following shows the performance results for each test case. Each test was run three times, each time was 0x8000 iterations. Optimization flags were tried for the C, but honestly I did not see much difference. Ruby was run on version 2.6.5p114.

|                                   | C with Regex - Multiple | C with Regex - once | C without Regex | Ruby with Regex - Multiple #scan | Ruby with Regex - Once #match |
| :-------------------------------- | :---------------------- | :------------------ | :-------------- | :------------------------------- | :---------------------------- |
| 1st Attempt                       | 554.1 ms                | 78.42 ms            | 49.65 ms        | 206.4 ms                         | 150.9 ms                      |
| 2nd Attempt                       | 560.3 ms                | 78.48 ms            | 47.79 ms        | 191.1 ms                         | 150.1 ms                      |
| 3rd Attempt                       | 569.5 ms                | 76.52 ms            | 47.77 ms        | 189.6 ms                         | 149.5 ms                      |
| Estimated Max Requests per Second | 2 requests              | 13 requests         | 21 requests     | 5 requests                       | 7 requests                    |

## Thoughts

Out of the data I collected, here are the thoughts I have:

* Performance says not using regular expression is better than using it.
* If regular expression is necessary, manually manage pattern compilation whenever appropriate, because this seems to be a high source of performance penality. For example:
  * In C, run regcomp separately, and then pass regex_t as an argument.
  * In Ruby (and I'm sure other languages apply), prepare Regexp manaully, and pass it as an argument.
* Ruby may be a slower choice, but a poorly implemented algorithm in C would not beat it either. This is like saying a slower sports car beats a faster sprts car, because the driver is better.

And finally, to answer my original question: Assuming regular expression provides enough security, how much performance am I asking the developers to sacrifice, and is that worth it?

I think my answer is, depending on the scale, everyone may be able to get away with it, or it could be catastrophic. When suggesting a security fix, perhaps it's better to leave the answer generic and give the developers the freedom to approach the fix. If an example of a fix should be provided, take the time to understand the codebase to get more context, consider the design goals, and advice accordingly.