
# **Documentation as Code**

### Introduction

With nowadays virtualization technologies, low latency communications, CPU Power and The Cloud, the Infrastructure paradigm is being changed from the static old-fashion way of managing servers to a new standard automation way of deploying services.

Sysadmins are becoming SysDevs and the widely famous DevOps term is starting to be defined by itself. Of course, the old-fashion way is still there though – and always from my humble point of view – any company with growth objectives will need to start changing the way of doing things in less than five years and start to automate their infrastructure.

### Definition – Still a new concept?

When typing “Infrastructure as Code” in any Search Provider, about 350K results will be returned including a [Wikipedia](https://en.wikipedia.org/wiki/Infrastructure_as_code) entry. New books are being published about this subject and it is clear that there is a new tendency when deploying new servers and services.

There are several approaches when talking about code development techniques and documentation like Wikis, Git Markdown, documentation along with the source, etc.

But what about doing it altogether? What about doing Documentation as Code?

Documentation as Code is the way of automating documentation along with the written code. It can be when deploying new Infrastructure elements, new services or just a new piece of software or script. Automated Documentation will be able to be exported and shared using external tools like Wikis, Content Management, Git Pages or others.


### The Idea

#### Bash Example

Let’s suppose that we want to create a bash script. Our documentation will be together with our script code. The difference is that it will be able to be exported with a flag given. This flag will also allow the exporting of the document in markdown and even in HTML or PDF.

```bash
#!/usr/bin/env bash
# Example of Documentation as Code

if [ $# -eq 0 ]; then
 echo "$(basename $0) [run|doc]"
fi

case $1 in
 run)

echo "Here goes the code to run"
 exit
 ;;

doc)

sed '/^:/,/^DAC/!d;s/^:/cat/' "$0" | bash -s "$@"
 exit
 ;;

*) exit
 ;;
esac


: <<DACv1
## Documentation Starts

Here you can add all the needed markdown documentation. You can split it as shown in this script.
DACv1

: <<\DACv2
### Documentation Continues
Here goes the documentation example with a table:

1 | 2 | 3 | 4 | 5
---|---|---|---
T | H | I | S |
I | S |   | A |
T | A | B | L | E
DACv2
```

#### Python Example

Going on with the same approach, we can do the same with other scripting languages. Here it is my idea with Python.

```python
#!/usr/bin/env python
# Example of Documentation as Code
import optparse

def main():
    parser = optparse.OptionParser(usage='Usage: %prog -o [run|doc]')
    dac_choices = ['run', 'doc', 'Run', 'Doc', 'RUN', 'DOC'] 
    parser.add_option("-o", dest="runordoc",
                      type="choice",
                      choices=dac_choices,
                      help="Please specify if you want Run or Doc")

    (opts, args) = parser.parse_args()

    if opts.runordoc is None:
        print "A mandatory option is missing\n"
        parser.print_help()
        exit(-1)

    if opts.runordoc == "Run" or opts.runordoc == "run" or opts.runordoc =="RUN":
        print "Here goes the code to run"
        exit(0)

    elif opts.runordoc == "Doc" or opts.runordoc == "doc" or opts.runordoc =="DOC":
        DACv1 = """## Documentation Starts

Here you can add all the needed markdown documentation. You can split it as shown in this script.

"""

        DACv2 = """### Documentation Continues

Here goes the documentation example with a table:

 1 | 2 | 3 | 4 | 5
---|---|---|---
 T | H | I | S |
 I | S |   | A |
 T | A | B | L | E
"""

        print DACv1
        print DACv2.strip()

if __name__ == '__main__':
    main()
```

#### PowerShell Example

The “Here-Strings” in PowerShell will do the job. Let’s see how it works.

```powershell
# Example of Documentation as Code

[CmdletBinding(SupportsShouldProcess=$True)]
Param(
  [Parameter(Mandatory=$true,Position=0, HelpMessage="Please specify if you want Run or Doc")]
  [ValidateSet("Run","Doc")]
  [string]$RunorDoc = $null
)

if ($PSCmdlet.ShouldProcess("$RunorDoc","Return Options"))

{
switch ($RunorDoc)
{
    "Run" {Write-Output "Here goes the code to run `$RunorDoc = $RunorDoc"}

    "Doc" {

# DACv1    

@"
## Documentation Starts

Here you can add all the needed markdown documentation. You can split it as shown in this script.

"@

# DACv2

@"

### Documentation Continues

Here goes the documentation example with a table:

1 | 2 | 3 | 4 | 5
---|---|---|---
T | H | I | S |
I | S |   | A |
T | A | B | L | E
"@
                }
        }
}
```

### Final Step

From here, we can use tools like Pandoc or AsciiDoctor inside our scripts to automatically convert and post our documentation to a Wiki, GitHub, Content Management, etc. Another approach is keeping the markdown format and using Jekyll to automatically creating amazing documentation portals.


## Conclusion

Documentation as Code is still a raw concept and I’m sure that it will start to emerge in a few years or months. To me, it is a procedure together with tools to get the best documentation and revision control along with the code. The way of creating documentation must be simple and the result must be useful.  The scripts shown here are just ideas and I’m sure that once you read this, a lot of new ones will strike your head.

!!! note
    These docs are created for my personal use. You are welcome to read them.