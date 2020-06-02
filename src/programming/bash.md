# Bash

## Special characters in file names

`-` is recognized as stdin, so use `./-` for checking filenames beginning with `-`.

Use `"hello world"` when names contain spaces.

## IO redirection

**Note**: Basic file descriptor (fd) for shell's operation are 0 for STDIN, 1 for STDOUT, 2 for STDERR.

I used this code for [OverTheWire Bandit 25](https://overthewire.org/wargames/bandit/bandit25.html).

```bash
#!/bin/bash

# Link fd3 to localhost:30002 bidirectional
exec 3<>/dev/tcp/localhost/30002	
# Print out first line from fd3
head -1 <&3							

old_password="UoMYTrfrBFHyQXmg6gzctqAwOmw1IohZ"

for i in {0000..9999}
do
    answer="${old_password} ${i}"
    # Send to fd3
    echo $answer >&3				
    response=$(head -1 <&3)
    if ! [[ $response =~ "Wrong" ]]
    then
        echo $i
        break
    fi
done

# Reset fd3 to normal
exec 3>&-
```

# Variables

No spaces between `=`, ie. `x=23`.
String interpolation uses `${}`.

```bash
x="hello"
y="world"
z="${x} ${y}"
```

# netcat

Use `nc` to create a server/client to send tcp.

```bash
# Server
nc -l 1234
# Client
nc localhost 1234
```