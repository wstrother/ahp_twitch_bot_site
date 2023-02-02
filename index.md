---
permalink: /
layout: page
title: index
---

## Introduction

The creaively named "AHP Twitch Bot" is a Python repository that I made which allows you to deploy a Twitch bot to enhance your streaming experience.
The implementation of this bot relies on a JSON settings file with a particular specification for defining commands, state variables, and privileged users
in order to create your own custom functionality. This webpage serves as a documentation for that specification.

This documentation resource does not presume any programming knowledge on the part of the reader. I may refernce the Python implementation with regards
to a specific functionality, but this is mainly for completeness and it's okay if you don't understand it. If you are interested in the implementation details
you can always view the source code in the actual repo, which aims to have complete doc strings and extensive comments for additional documentation.

#### Requirements

* [Python 3](https://www.python.org/downloads/)
* [A Twitch account for your bot](https://www.twitch.tv/signup)

#### Resources

* [JSON](https://developer.mozilla.org/en-US/docs/Glossary/JSON)
* [Python](https://docs.python.org/3/)
* [AHP Twitch Bot on Github](https://github.com/wstrother/ahp_twitch_bot)

---

## Getting Started

AHP Twitch Bot is implemented in Python and to run it locally on your computer you will need to have Python installed, but you don't need to know anything
about programming in order to use it to enhance your streaming experience. The following files are all you need to get started.

### **stream.py**

This file represents the application entry point and an example is provided in the core repository. Once you have cloned the repository or downloaded
and extracted it as a zip file, you can change the file `stream.py.example` to be named `stream.py` and edit it as follows:

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

**Note:** It should go without saying, but don't create a bot and tell it to connect to someone else's stream chat unless they specifically asked you to. The Twitch IRC
protocol that this bot makes use of is totall open and would technically allow you to do this because it's literally the same as joining someone's chat through
the Twitch website. But it would be very rude to do this, and you can be banned, and it most certainly violates Twitch's terms of service to use an automated program
to send messages in someone's chat without their permission!

Once you have edited `stream.py` to allow your Twitch bot's account to connect your chat, you can run the program by opening a command prompt window in the folder
where you cloned/extracted the repository and typing the following command:

```json
> python stream.py
```

### oauth.token

Be sure to create a file for your oauth token (obtainable [here](https://twitchapps.com/tmi/)). Simply copy paste the token into the file and save it.

***REMEMBER TO KEEP THIS TOKEN SECRET!***

It represents a tokenized form of your authentication with Twitch and the authorization to connect to a Twitch IRC channel (i.e. a "Twitch Chat" for a given user). 
If you ever accidentally share this token, you can revoke it or generate a new token (rendering the old one invalid) from your
[connection settings page](https://www.twitch.tv/settings/connections) on Twitch.

### **bot_settings.json**

To define attributes and behaviors for your bot, you'll need to create a JSON data file and provide some instructions. The following keys can be defined to support
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

* **approved_users:** defines user names for users that are allowed
to use restricted commands
* **restricted:** defines commands that can only be used by approved
users
* **public:** defines commands that anyone in chat can use
* **state:** defines a set of arbitrary data and variables that can be set or referenced by certain commands

In the example JSON above, you will see I have used *:command* to represent the individual commands that will be defined in this file, but note that
this is just a shorthand for the JSON specification that will be defined in the following sections.

---

## Defining Commands

Commands defined within your JSON data will need to be expressed as an array that contain a series of parameters. This array will always begin by specifying 
a CommandClass, then the *name* of the command, (i.e.: how it is invoked in chat) and then any initialization parameters.

**Note**: The terms "parameter" and "argument" are often interchangeable, but here I use "parameter" to refers specifically to the "initialization arguments" of the command itself.

In the Python implementation, these are literally the arguments passed to the `CommandClass.__init__()` method when the class object is instantiated. 
In other words: they are the settings that define what the command actually does.

This usage can be confusing because -- like with other Twitch Bots -- the command can also be thought of as taking "arguments" when invoked by a user in chat,
so I will try to be very consistent in using "parameter" and "argument" to mean different things.

A command can be invoked in chat by typing *!name_of_command* and the "arguments" passed to it are just any following words, separated by spaces, 
i.e.: *!name_of_command argument1 argument2...* and so on. 

For example,
I might have a command that viewers can invoke called *!pb* to see my current personal best in a given speed game. The user might type *!pb alttp* to see
my current personal best in 'The Legend of Zelda: A Link to the Past.' In this case, *ssb* is an "argument" for the command's usage,
not the command itself or its definition.

Again, in the Python implementation, this sense of "argument" is passed to the `CommandClass.do()` method. If you're unsure what that means, the important
point to take away is that **arguments passed to commands by users in chat DO NOT permanently alter the functionality of the command itself.** They only affect
what it does in that particular case.

#### JSON Specification for Commands

In the example JSON provided above, the use of *:command1* etc. is a shorthand for the following syntax:

```json
["CommandClass", "command_name", "parameter1", "parameter2"...]
```

In JSON terms, this is an array of strings whereby the first string specifies the particular type of command you are defining (See below). The second string
specifies the name of the command, which to say, how it is invoked in chat, i.e.: *!command_name*

The parameters will depend on the type of command you are defining and what you want it to do. 

---

## CommandClasses

I've used the term "`CommandClass`" throughout this document to refer to (in Python terms) subclasses of the base class `Comand` as defined in `commands.py` in the source code. 
The names of these subclasses (with the CamelCase convention, as listed below) are the literal strings you use to specify what type of command you are creating.

The following subclasses define the types of commands you can create, but as you will see, their combined usage can get quite complicated and powerful
because of the way they can be joined together and used to invoke each other. We'll start with the simplest examples and work our way up.


### TextCommand

```json
["TextCommand", "name", "Text for your bot to send to the chat"]
```

The simplest use case, this type of command simply causes your bot to say something to the chat whenever it is invoked.

### FormatCommand

```json
["FormatCommand", "name", "Formatted text with {variable_interploation}"]
```

Similar to the `TextCommand` class, these commands support a type of variable interpolation based on the specification in Python's
`str.format()` method. The set of variables passed to this formatting method come from the bot's state variables
and they can be interpolated into the string by enclosing the variable name in curly brackets like this: *'{variable_name}'*

**Note:** These variable interpolations can also support key or index specifications within square brackets, i.e.: *'{variable[subkey]}'*. So if, for example,
you have a state variable called `list` that looks like *["One", "Two", "Three"]* then using the formatting parameter *"{list[0]}"* 
your bot will send the message *One* to the chat.

Python's string formatting method also supports some additional formatting arguments as specified [here.](https://docs.python.org/3/library/string.html#format-specification-mini-language)
This allows for padding, alignment, and some numeric conversion. Bear in mind that in the source code's implementation
the formatting method is called with the state variables passed as named keywords, so the `FormatCommand` class itself doesn't
support the use of positional arguments.

### JsonCommand

```json
["JsonCommand", "name", {"some_key": "some value that can be {variable_interpolated}"}]
```

The `JsonCommand` class takes a third parameter that should be a JSON object rather than a string. The intended use for this
`CommandClass` is to be able to create a JSON formatted string that can then be passed to a `PostCommand` to make up the body of its post request.

They value for each key within the JSON object can also be interpolated with state variables, just like the FormatCommand class.
This formatting is performed recursively so strings within nested objects will also be interpolated.


### PostCommand

```json
["PostCommand", "name", "url"]
```

The PostCommand is used to send a POST request to an API endpoint specified by the URL parameter.

Note that when the `PostCommand` is invoked in chat, the argument passed to it must be a valid JSON serializable string, which is treated as
the body of the POST request. Because this is error prone and unwieldy, it's reccomended that the `PostCommand` be used in conjunction 
with other commands, typically a `JsonCommand` in order to provide a properly fromatted JSON serializable string.

### GetCommand

```json
["PostCommand", "name", "url"]
```

Similar to the `PostCommand`, the `GetCommand` sends a GET an API endpoint specified by the URL parameter and echos its output to chat.

Because most web APIs return data as serialized JSON, it might be more useful to chain this command with other commands, just like the `PostCommand`.


### AliasCommand

```python
["AliasCommand", "name", "other_command", "arg1", "arg2"...]
```

These commands do not take any arguments. Instead they invoke some other command with a specified set of arguments passed to it. 
They are useful for creating a shorthand for a common command/argument pairs such as typing *!ssb* to invoke *!game Super Smash Bros.*.

### StateCommand

```python
["StateCommand", "name", (optional)"key"]
```

These commands set some 'state variable' of the bot. By default, the name of the command is used as the name for the state variable,
 but the optional third parameter can be specified so that the command's name is different from the name of the state variable.

### SubStateCommand

```python
["SubStateCommand", "name", "variable_name", "variable_key"]
```

Similar to the `StateCommand`, the `SubStateCommand` can be used to set the value of some key within an object, or index of an array
that isstored as a state variable. Note that arrays stored as state variables are 0 indexed, so to return the first value of an array
the number 1 should be used as the final parameter.

**Note:** there is no `SubSubStateCommand` class, so there is inherently a depth limit of 1 for key/index values that can be accessed
by using any command. Nothing prevents you from storing state variables with more complex nesting, but as a general rule it should
be avoided because of this limitation.

### ChainCommand

```json
["ChainCommand", "name", "output_command", "input_command"]
```

The ChainCommand is used to pass the "output" (i.e.: any message that would be sent to chat) of one command to the "input" of another
(i.e.: the arguments that would be passed if the command were invoked in chat).

This command is particularly powerful when combined with `SequenceCommands` and `OptionCommands`, but another common use case would be
to chain the output of a `JsonCommand` to the input of a PostCommand in order to, for example, use a set of state variables to make up
the body of a POST request for an external API.

### SequenceCommand

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
* **command_name":** the name of some other command to invoke
* **["command_name", "arg1", "arg2"...]:** the name of some other
command to invoke and a set of arguments to pass to it before the
arguments invoked with the original SequenceCommand

Because usage of these expressions is so flexible, it's important to be
clear on the way arguments specified in these expressions interact with
the arguments passed from the chat as the SequenceCommand is invoked.

In the first case, the *init_args* are used to instantiate the anonymous
command, and arguments passed to the SequenceCommand will also be passed to
that anonymous command.

In the second case, the same arguments passed to the SequenceCommand are
passed to the named comman specified for that step.

In the third case, the arguments specified in the *:command_entry* step
expression are passed *before* the arguments invoked with the SequenceCommand
in the chat. So typing *!sequence_command arg3 arg4* where *sequence_command*
has some step expressed as **['other_command', 'arg1', 'arg2']** would be the
same as typing *!other_command arg1 arg2 arg3 arg4*.

### OptionCommand

```python
["OptionCommand", "name"
  :command_entry1,
  :command_entry2...
]
```

These commands take some argument and use it to determine which from a
series of other commands to invoke, passing all arguments after the first
to that other command. Here, any *:command_entry* expression can have
any of the same formats specified for **SequenceCommand**, including
'anonymous' commands and other commands invoked with specified arguments.

The same rules apply for passing arguments as with **SequenceCommand**
except that the first argument passed to the initial **OptionCommand**
will not be passed to the option selected. So *!option_command choice
arg1* will not pass *choice* to the other command specified by that
choice.

