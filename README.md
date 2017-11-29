# Logging - Best Practices (Draft)

## In The "old" days ...

... more often than not, developers treated logs as some kind of an afterthought. While hammering out line after line of code, adding functionality and exstinguishing errors, it was THEM and THE DBUGGER (ok, sometimes printf).

... only afterwards, when a program misbehaved in its **production** environment, THE DEBUGGER was no longer an option. The only hope was (and is) that the application provided sufficient logging functionality.

## Today ...

... in the era of devops, log information and other machine data (e.g. runtime metrics, ...) are even more important when combined with automatic processing, analytics and alerting for service/support and product development. 

## Log types

There are at least two kinds of logs:

- Application logs

    Information of different level about state, behaviour and usage of the application for service, support and development. 

> Beware of data/information with **special legal/technical security requirements**. Most application logs can be read by any user/person with access to the system the application is running on.

- Audit logs
    
    Detailed information about application events and actions for legal or security scrutiny. 

> An audit log is most likely an immutable protocol **requiring special permissions** and may contain ANY kind of information required to verify executed actions.
     
## Use The ~~Force~~ Library, Luke

Anyone can write text to the console or a file, _but this is not exactly logging_.

At least, beside the text message, **some ordering info (most likely a timestamp and something more)** is required.

With that said, quite a lot of decisions are to be made:

- when to start or stop logging
- how to display only the required information
- what is the optimal solution to minimize performance impact 
- where to place ordering info
- which language to choose
- what format should the log be
- which time resolution/timezone to use
- should the information only end up on the console, the file or ...
- ... 

# THE best advice/practice

> If possible, use an existing logging library. Really!

Based on the chosen library and its configuration, the log library ...

- takes care of the ordering problem by adding information like time, process-id, thread-id, application-name, etc.
- provides structured layout of the information output
- supports differents output formats like xml, json, csv, ...
- filters log information according to severity levels
- might even handle multiple targets

# Other best practices

## What to log ...

Events, states and metrics that help by identifying problems or verify regular/expected behaviour.

Examples of user/admin messages (usually to stdout)

- Application started
- Opened database xyz
- Could not write file. path="c:\..."
- Configuration updated. User="someone"

Examples of developer messages (usually to stderr)

- [Database.cs:4711][Handler]NullPointer exception ...

## What NOT to log ...

Do **NOT log business data** in an application log as plaintext (you might be required to do so in an audit log).

BAD Examples:

- Transferring 1.000€ using Visa 1234-1234-1234-1234 123 of FirstName LastName ...
- User LastName, FirstName accessed web page "https://bad.content.org"

Better/Good:

- Transferrig money. Amount: €1000, CredentialsId: "4ee6608f-09c0-404d-93b1-65a8b398bc32"
- Web page accessed. User: "user0071", UrlHash: "(sha1)cf23df2207d99a74fbe169e3eba035e633b65d94"

## Treat the log as "a stream"

In the [The twelve-factor app](https://12factor.net/), logging is mentioned as ["Treat logs as event streams"](https://12factor.net/logs).

A stream has no start, no end and can be forwarded to "somewehere else" (logs are simply being "continued"...).

For containers et,c, log messages are written to stdout and/or stderr and handled by the runtime environment from there.

## Where do log messages finally go to

Logs can "stay on the system" (file, db, event-log, ...), be persisted on some server (e.g. database) or handed over to some service (API).

Any solution (local database, windows event log, syslog, ...) must be able to export or send logs to other destinations.

> From ["Treat logs as event streams"](https://12factor.net/logs): Most significantly, the stream can be sent to a log indexing and analysis system [...] 

## Log solutions (transfer, storage and access implications)

As mentioned before, logs might contain information with specific legal/technical restrictions for access, location etc.

**Very carefully review** any log solution for technical and legal requirements to 
- transport (w/o encryption)
- content (masking)
- storage (w/o encryption)
- access (permission, roles, ...)
- location (continent, country, building, ...)


## Log configuration

### Use UTC for logging

The creation, processing and display of related log information can (and will) happen at different locations.

It doesn't make sense to store time information as "local time" (with the occasional daylight saving problem). 

> It is of utmost importance to establish a common timebase. Using [UTC](https://www.timeanddate.com/worldclock/timezone/utc) for logging everywhere makes things much easier.    

### Use Internet/Network time for logging

Avoid using incorrect time by incorporating a local timeserver in your network or a well known timeserver on/from the internet.   


## How to format/structure log messages (or: prepare for parsing).

Inside the application, a log message is basically triggered by a function or a method controlled by a __formatting message__.

Here's some hypothetical logging statement:

        logger.Trace("User {0} plays {1} in game {2}", userId, card, gameId);    

This creates something like:

        2013-01-12 17:49:37,656 [T1] INFO  c.d.g.UserRequest  User 1334563 plays 4 of spades in game 23425656

Automatic processing of the log message using regex (message part only) requires something like:

        /User (\d+) plays (.+?) in game (\d+)$/
        
>Regex are tricky, hard to decipher and depend on the grammar of the language (english, german, ...). Especially think about data with spaces or other non-alphanumerical content ...

### Option 1: Use a "native" format 

In combination with a library supporting json format(ting) and a different __formatting message__ extracting the data from the log becomes much easier:

        logger.Trace("{info: 'User plays', user: {0}, card: '{1}', game: {2}}", userId, card, gameId);    

or (if the hypothetical library supports serialization):

        msg = { info: 'User plays', user: 1334563, card: '4 of spades', game: 23425656 };
        logger.Trace(msg);    

This can be 

        { timestamp: '2013-01-12 17:49:37,656', tid: 'T1', severity: 'INFO', method: 'c.d.g.UserRequest', message: { info: 'User plays', user: 1334563, card: '4 of spades', game: 23425656 } }

And the rest is left to some JSON parser ...


### Option 2: Structured log messages

If the library in use is not able to produce pure "native" formats, automatic log processing still picks up speed with EASY to parse messages. With T=timestamp, I=id, L=level, C=classname and N=V as name=value pairs, just imagine a log format similar to this

        TTTTTTTTTTTTTTTTTTTTTTT IIII LLLL  CCCCCCCCCCCCCC  N=V,...,N=V
 
        2013-01-12 17:49:37,656 [T1] INFO  c.d.g.DataPath  action=Data path found in registry, Path=C:\ProgramData\Xxxxxxx\xxxx + xxxxxx\Xxxxx\, Result=0x0
        2013-01-12 17:49:37,884 [T1] INFO  c.d.g.RegistryOpen  action=Read registry, Key=\\HKLM\..., Result='Safe', Code=0x0

Such lines (the important name=value bits) can be parsed with a fairly simple regex (values MUST NOT contain ',')

        /([^ ][^=]*)=([^,]*)[ ,]*/g

regex results for examples

        Match 1
        Full match	0-87	`2013-01-12 17:49:37,656 [T1] INFO  c.d.g.DataPath  action=Data path found in registry, `
        Group 1.	0-57	`2013-01-12 17:49:37,656 [T1] INFO  c.d.g.DataPath  action`
        Group 2.	58-85	`Data path found in registry`
        
        Match 2
        Full match	87-137	`Path=C:\ProgramData\Xxxxxxx\xxxx + xxxxxx\Xxxxx\, `
        Group 1.	87-91	`Path`
        Group 2.	92-135	`C:\ProgramData\Xxxxxxx\xxxx + xxxxxx\Xxxxx\`
        
        Match 3
        Full match	137-147	`Result=0x0`
        Group 1.	137-143	`Result`
        Group 2.	144-147	`0x0`


        Match 1
        Full match	0-22	`action=Read registry, `
        Group 1.	0-6	    `action`
        Group 2.	7-20	`Read registry`
        
        Match 2
        Full match	22-38	`Key=\\HKLM\..., `
        Group 1.	22-25	`Key`
        Group 2.	26-36	`\\HKLM\...`
        
        Match 3
        Full match	38-53	`Result='Safe', `
        Group 1.	38-44	`Result`
        Group 2.	45-51	`'Safe'`
        
        Match 4
        Full match	53-61	`Code=0x0`
        Group 1.	53-57	`Code`
        Group 2.	58-61	`0x0`

        
>"Structured log messages" have to take care of required escaping (put an escape character like \ in front) or quoting (placing " or ' around data with spaces) for the data in question.


### Language(s)

English seems to be a good choice - at least for attribute __names__ etc.. 

_Bad_ (Application, queries etc. have to be adjusted to new language as well)

#### en
    { message: "Data path found in registry.", path: "C:\\ProgramData\\Xxxxxxx\\xxxx + xxxxxx\\Xxxxx\\", result: 0x0 }
    
#### de
    { Nachricht: "Datenpfad in der Registrierung gefunden.", Pfad: "C:\\ProgramData\\Xxxxxxx\\xxxx + xxxxxx\\Xxxxx\\", Resultat: 0x0 }

_Good_ (Only the displayed message changed)

#### en
    { message: "Data path found in registry.", path: "C:\\ProgramData\\Xxxxxxx\\xxxx + xxxxxx\\Xxxxx\\", result: 0x0 }
    
#### de
    { message: "Datenpfad in der Registrierung gefunden.", path: "C:\\ProgramData\\Xxxxxxx\\xxxx + xxxxxx\\Xxxxx\\", result: 0x0 }



## Better / more usefull logging

### Linked logs should support "correlation-ids"

If multiple system or applications work in concert, logs must provide hints how to "connect the dots" (where does one log message refer to a second one).


## Other/more best practices for logging (call it references :-D)

[Splunk:Logging best practices](http://dev.splunk.com/view/logging/SP-CAAAFCK)

[The 10 Commandments of Logging](http://www.masterzen.fr/2013/01/13/the-10-commandments-of-logging/)
