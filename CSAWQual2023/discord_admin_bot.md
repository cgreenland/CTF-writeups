# Discord Admin Bot

### Description

>Join discord and get the flag.
>discord.gg/csaw23 [repeated 9 times]

Provided file `bot_send.py` is as follows
```python
#############################################
# Author: Krishnan Navadia
# This is main working file for this chal
#############################################

import discord
from discord.ext import commands, tasks
import subprocess

from settings import ADMIN_ROLE
import os
from dotenv import load_dotenv
from time import time

load_dotenv()

TOKEN = os.getenv("TOKEN")

intents = discord.Intents.default()
intents.messages = True
bot = commands.Bot(command_prefix="!", intents=intents)

bot.remove_command('help')

SHELL_ESCAPE_CHARS = [":", "curl", "bash", "bin", "sh", "exec", "eval,", "|", "import", "chr", "subprocess", "pty", "popen", "read", "get_data", "echo", "builtins", "getattr"]

COOLDOWN = []

def excape_chars(strings_array, text):
    return any(string in text for string in strings_array)

def pyjail(text):
    if excape_chars(SHELL_ESCAPE_CHARS, text):
        return "No shells are allowed"

    text = f"print(eval(\"{text}\"))"
    proc = subprocess.Popen(['python3', '-c', text], stdout=subprocess.PIPE, preexec_fn=os.setsid)
    output = ""
    try:
        out, err = proc.communicate(timeout=1)
        output = out.decode().replace("\r", "")
        print(output)
        print('terminating process now')
        proc.terminate()
    except Exception as e:
        proc.kill()
        print(e)

    if output:
        return f"```{output}```"


@bot.event
async def on_ready():
    print(f'{bot.user} successfully logged in!')

@bot.command(name="flag", pass_context=True)
async def flag(ctx):
    admin_flag = any(role.name == ADMIN_ROLE for role in ctx.message.author.roles)

    if admin_flag:
        cmds = "Here are some functionalities of the bot\n\n`!add <number1> + <number2>`\n`!sub <number1> - <number2>`"
        await ctx.send(cmds)
    else:
        message = "Only 'admin' can see the flag.😇"
        await ctx.send(message)

@bot.command(name="add", pass_context=True)
async def add(ctx, *args):
    admin_flag = any(role.name == ADMIN_ROLE for role in ctx.message.author.roles)
    if admin_flag:
        arg = " ".join(list(args))
        user_id = ctx.message.author.id
        ans = pyjail(arg)
        if ans: await ctx.send(ans)
    else:
        await ctx.send("no flag for you, you are cheating.😔")

@bot.command(name="sub", pass_context=True)
async def sub(ctx, *args):
    admin_flag = any(role.name == ADMIN_ROLE for role in ctx.message.author.roles)
    if admin_flag:
        arg = " ".join(list(args))
        ans = pyjail(arg)
        if ans: await ctx.send(ans)
    else:
        await ctx.send("no flag for you, you are cheating.😔")


@bot.command(name="help", pass_context=True)
async def help(ctx, *args):
    await ctx.send("Try getting `!flag` buddy... Try getting flag.😉")


@bot.event
async def on_command_error(ctx, error):
    if isinstance(error, commands.CommandNotFound):
        await ctx.send("Try getting `!flag` buddy... Try getting flag.😉")
    else:
        print(f'Error: {error}')


bot.run(TOKEN)
```

### Solution

Having no experience with Discord bots, I thought this would be an interesting opportunity to learn something new, and I was right! Examining the code above, I could see the bot appeared to some commands !flag, !help, !add, and !sub, but all the commands except !help have their interesting parts guarded by an admin role check.

Right away we can see that there is some kind of input sanitization going on with a list of disallowed strings.

```python
SHELL_ESCAPE_CHARS = [":", "curl", "bash", "bin", "sh", "exec", "eval,", "|", "import", "chr", "subprocess", "pty", "popen", "read", "get_data", "echo", "builtins", "getattr"]
```

The most interesting part of the code is the `pyjail` function, which is called by the !add and !sub commands. It opens a subprocess to execute the console command `python -c print(eval({text}))`, where `text` contains the arguments from the !add or !sub and returns the output. We can see from the 3rd line that the developer is worried about preventing the user issuing shell commands, and with good reason, This is clearly opens up a command injection attack if we can get around the admin checks and input sanitization.

```python
def pyjail(text):
    if excape_chars(SHELL_ESCAPE_CHARS, text):
        return "No shells are allowed"

    text = f"print(eval(\"{text}\"))"
    proc = subprocess.Popen(['python3', '-c', text], stdout=subprocess.PIPE, preexec_fn=os.setsid)
    output = ""
    try:
        out, err = proc.communicate(timeout=1)
        output = out.decode().replace("\r", "")
        print(output)
        print('terminating process now')
        proc.terminate()
    except Exception as e:
        proc.kill()
        print(e)

    if output:
        return f"```{output}```"
```

Examining the !add command closely, we can see that it checks if the user issuing the command has the 'ADMIN_ROLE', which is defined from another file, only then does it call pyjail with our input arguments.

```python
@bot.command(name="add", pass_context=True)
async def add(ctx, *args):
    admin_flag = any(role.name == ADMIN_ROLE for role in ctx.message.author.roles)
    if admin_flag:
        arg = " ".join(list(args))
        user_id = ctx.message.author.id
        ans = pyjail(arg)
        if ans: await ctx.send(ans)
    else:
        await ctx.send("no flag for you, you are cheating.😔")

```

I couldn't see any way to bypass or overwrite the role check, and it seemed obvious there was no way to obtain this role on the CSAW Discord server. That seemed to leave only one possibility. Could I get the bot on a server I control? Would it be the same bot with access to the flag? I realized I needed to do some research to confirm my ideas, so I started googling for past CTF writeups involving Discord bots. I quickly stumbled on [this writeup](https://ctftime.org/writeup/33674), which indeed showed how to invite a public bot to your own server using an invite link. Following the steps in that writeup, I created the invite link below, which I pasted in my browser.

`https://discord.com/oauth2/authorize?client_id=1152454751879962755&permissions=8&scope=bot`

It worked! The bot was in my server!

![](../images/discord_join.png)

Based on the helpful message, I surmised that `ADMIN_ROLE` was simply `admin` and gave myself this role. Now I had access to the commands.

Attempting the obvious, issuing commands like `!add import('os').system('whoami')` were filtered by the input sanitization list and bot responded with `No shells are allowed`.

Some googling later, I found [this page on bypassing pyjails](https://book.hacktricks.xyz/generic-methodologies-and-resources/python/bypass-python-sandboxes), the section on [eval-ing Python code](https://book.hacktricks.xyz/generic-methodologies-and-resources/python/bypass-python-sandboxes#eval-ing-python-code) was most helpful. The first thin I tried that worked was the octal encoding of `__import__('os').system('ls')`

`!add \137\137\151\155\160\157\162\164\137\137\50\47\157\163\47\51\56\163\171\163\164\145\155\50\47\154\163\47\51`

Progress! Now changing the Python command to `__import__('os').system('cat flag.txt')` and encoding in octal I can issue

`!add \137\137\151\155\160\157\162\164\137\137\50\47\157\163\47\51\56\163\171\163\164\145\155\50\47\143\141\164\40\146\154\141\147\56\164\170\164\47\51` 

Which gives the flag: `csawctf{Y0u_4r3_th3_fl4g_t0_my_pyj4il_ch4ll3ng3}`
