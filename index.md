---
permalink: /
layout: page
title: index
---

## Introduction

The creaively named "AHP Twitch Bot" is a Python repository that I made which allows you to deploy a Twitch bot to enhance your streaming experience.
The implementation of this bot relies on a JSON settings file with a particular specification for defining commands, state variables, and privileged users
in order to create your own custom functionality. This webpage serves as a documentation for that specification.

This documentation resource *does not presume any programming knowledge* on the part of the reader. I may refernce the Python implementation with regards
to a specific functionality, but this is mainly for completeness and it's okay if you don't understand it. If you are interested in the implementation details
you can always view the source code in the actual repo, which aims to have complete doc strings and extensive comments for additional documentation.

### Requirements

* Python 3
* A Twitch account for your bot

---

## Repository Structure

Any of the classes defined in the four core modules can be subclassed to define more specific functionality, but the **BotLoader** class in **twitch_bot.py** 
helps define a JSON based data interface that can be used to automatically customize the implementation of your Twitch bot. If you aren't comfortable with Python
or don't require any extensive custom functionality, the following files are the only ones you need to worry about.

### **stream.py**

This file represents the application entry point and an example is provided in the core repository.

```python
from ahp_twitch_bot.twitch_bot import TwitchBot, BotLoader

if __name__ == "__main__":
  USER_NAME = "your_bot_user_name"
  CHANNEL = "#your_twitch_channel"
  TOKEN = "oauth.token"
  SETTINGS = "bot_settings.json"
  JOIN_MESSAGE = "Logging on..."

  BotLoader.load_bot(
      SETTINGS, TwitchBot, (USER_NAME, TOKEN)
  ).run(CHANNEL, JOIN_MESSAGE)
```

The **USER_NAME** and **CHANNEL** variable should be set to the user name of the Twitch account you created for your bot, and the channel you want it to connect to,
i.e. your own Twitch user name. 

### oauth.token

Be sure to create a file for your oauth token (obtainable [here](https://twitchapps.com/tmi/)). 

**REMEMBER TO KEEP THIS TOKEN SECRET!** 

It represents a tokenized form of your authentication with Twitch and the authorization to connect to a Twitch IRC channel (i.e. a "Twitch Chat" for a given user). 
If you ever accidentally share this token, you can revoke it or generate a new token (rendering the old one invalid) from your
[connection settings page](https://www.twitch.tv/settings/connections) on Twitch.

### **bot_settings.json**

To define attributes and behaviors for your bot, you'll need to create a JSON data file and provide some instuctions. The following keys can be defined to support
whatever functionality you want to implement.

```json
{
  "approved_users": [
      "user_1",
      "user_2",
      ...
    ],
  "restricted": [
      :command1,
      :command2,
      ...
    ],
  "public": [
      :command3,
      :command4,
      ...
    ],
  "state": {
    "variable1": "initial value",
    "variable2": "some other initial value",
    ...
  }
}
```

* **"approved_users":** defines user names for users that are allowed
to use restricted commands

* **"restricted":** defines commands that can only be used by approved
users

* **"public":** defines commands that anyone in chat can use

* **"state":** defines a set of arbitrary data and variables that can be set or referenced by certain commands

In the example JSON above, you will see I have used *:command* to represent the individual commands that will be defined in this file, but note that
here I used *:command* as a shorthand for the JSON specification that will be defined in the following sections.

---

### Defining Commands

Commands defined within your JSON data will need to be expressed as an array that contain a series of arguments. This array will always begin by specifying 
a CommandClass, then the *'name'* of the command, (i.e.: how it is invoked in chat) and then any initialization parameters.

**Note**: The terms "parameter" and "argument" are often interchangeable, but here I use "parameter" to refers specifically to the "initialization arguments" of the command itself.

In the Python implementation, these are literally the arguments passed to the **CommandClass.__init__()** method when the class object is instantiated. 
In other words: they are the settings that define what the command actually does.

This usage can be confusing because -- like with other Twitch Bots -- the command can also be thought of as taking "arguments" when invoked by a user in chat,
so I will try to be very consistent in using "parameter" and "argument" to mean different things.

A command can be invoked in chat by typing *'!name_of_command'* and the "arguments" passed to it are just any following words, separated by spaces, 
i.e.: *!'name_of_command argument1 argument2...'* and so on. 

For example,
I might have a command that viewers can invoke called *'!pb'* to see my current personal best in a given speed game. The user might type *"!pb alttp"* to see
my current personal best in 'The Legend of Zelda: A Link to the Past.' In this case, *'ssb'* is an "argument" for the command's usage,
not the command itself or its definition.

To be clear, in the Python implementation, this sense of "argument" is passed to the **CommandClass.do()** method. If you're unsure what that means, the important
point to take away is that **"arguments" passed to commands by users in chat DO NOT permanently alter the functionality of the command itself.** They only affect
what it does in that particular case.

### JSON Specification for Commands

In the example JSON provided above, the use of *:command1* etc. is a shorthand for the following specification:

```json
["CommandClass", "command_name", "parameter1", "parameter2"...]
```

In JSON terms, this is an array of strings whereby the first string specifies the particular type of command you are defining (See below). The second string
specifies the name of the command, which to say, how it is invoked in chat, i.e.: *'!command_name'*

The parameters will depend on the type of command you are defining and what you want it to do. 

---

## CommandClasses

I've used the term "CommandClass" throughout this document to refer to (in Python terms) subclasses of the base class **Comand** as defined in **commands.py** 
in the source code. The names of these subclasses (with the CamelCase convention, as listed below) is the literal string you use 
to specify what type of command you are creating.

The following 10 subclasses define the types of commands you can create, but as you will see, their combined usage can get quite complicated
because of the way they can be combined and used to invoke each other. We'll start with the simplest examples and work our way up.


#### TextCommand

```json
["TextCommand", "name", "text for your bot to send to the chat"]
```

The simplest use case, this type of command simply causes your bot to say something to the chat whenever it is invoked.


#### AliasCommand

```python
["AliasCommand", "name", "other_command", "arg1", "arg2"...]
```

These commands do not take any arguments. Instead they invoke some other command with a specified set of arguments passed to it. 
They are useful for creating a shorthand for a common command/argument pairs such as typing *"!ssb"* to invoke *"!game Super Smash Bros."*.


#### SequenceCommand

```json
["SequenceCommand", "name",
  :command_entry1,
  :command_entry2...
]
```

These commands will invoke a series of other commands, passing any arguments
they receive to each command in the series they invoke. Each ':command_entry'
item can be expressed in a number of ways:

* **["ClassName", "init_arg1", "init_arg2"...]:** an "anonymous" command,
which will not have a name or be invokable on it's own, and a set
of initialization arguments for that class
* **"command_name":** the name of some other command to invoke
* **["command_name", "arg1", "arg2"...]:** the name of some other
command to invoke and a set of arguments to pass to it before the
arguments invoked with the original SequenceCommand

Because usage of these expressions is so flexible, it's important to be
clear on the way arguments specified in these expressions interact with
the arguments passed from the chat as the SequenceCommand is invoked.

In the first case, the *'init_args'* are used to instantiate the anonymous
command, and arguments passed to the SequenceCommand will also be passed to
that anonymous command.

In the second case, the same arguments passed to the SequenceCommand are
passed to the named comman specified for that step.

In the third case, the arguments specified in the *':command_entry'* step
expression are passed *before* the arguments invoked with the SequenceCommand
in the chat. So typing *'!sequence_command arg3 arg4'* where *'sequence_command'*
has some step expressed as **['other_command', 'arg1', 'arg2']** would be the
same as typing *'!other_command arg1 arg2 arg3 arg4'*.

#### OptionCommand

```python
["OptionCommand", "name"
  :command_entry1,
  :command_entry2...
]
```

These commands take some argument and use it to determine which from a
series of other commands to invoke, passing all arguments after the first
to that other command. Here, any *':command_entry'* expression can have
any of the same formats specified for **SequenceCommand**, including
'anonymous' commands and other commands invoked with specified arguments.

The same rules apply for passing arguments as with **SequenceCommand**
except that the first argument passed to the initial **OptionCommand**
will not be passed to the option selected. So *'!option_command choice
arg1'* will not pais *'choice'* to the other command specified by that
choice.

#### StateCommand

```python
["StateCommand", "name", (optional)"key"]
```

#### SubStateCommand
#### FormatCommand
#### JsonCommand
#### ChainCommand
#### PostCommand

These commands set some 'state variable' of the bot, stored in the
**TwitchBot.state** dict attribute. The name of the command is used
as the key for the state dict by default, but optionally, the key
can be specified so that the command's name is different from the
name of the state variable.

The arguments passed to a **StateCommand** are concatenated by " "
strings such that *'!state_command some value here'* set's the state
variable to the string *'some value here'*.

