---
permalink: /
layout: page
title: index
toc: true
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
about programming in order to use it to enhance your streaming experience. 

First, start by cloning the repository (or downloading as a zip and extracting it). Now the following files are all you need to edit to get started.

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
protocol that this bot makes use of is totally open and would technically allow you to do this because it's literally the same as joining someone's chat through
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

In the example JSON above, you will see I have used `:command1` etc. to represent the individual commands that will be defined in this file, but note that
this is just a shorthand for the JSON specification that will be defined in the following sections.

**Note:** Your bot's state values are not persistent and if the `stream.py` process is terminated then upon reconnecting they will go back to the initial
values as defined in your setting file.

---

## Defining Commands

Commands defined within your JSON data will need to be expressed as an array that contain a series of parameters. This array will always begin by specifying 
a `CommandClass`, then the name of the command, and then any initialization parameters.

#### "Parameters" vs. "Arguments"

The terms "parameter" and "argument" are often interchangeable, but here I use "parameter" to refer specifically to the "initialization arguments" of the command itself.

In the Python implementation, these are the arguments passed to the `CommandClass.__init__()` method when the class object is instantiated. 
In other words: they are the settings that define what the command actually does.

This usage can be confusing because -- like with other Twitch Bots -- the command can also be thought of as taking "arguments" when invoked by a user in chat,
so I will try to be very consistent in using "parameter" and "argument" to mean different things.

A command can be invoked in chat by typing *!name* and the "arguments" passed to it are just any following words, separated by spaces, 
i.e.: *!name argument1 argument2...* and so on. 

For example,
I might have a command that viewers can invoke with *!pb* to see my current personal best in a given speed game. The user might type *!pb alttp* to see
my current personal best in 'The Legend of Zelda: A Link to the Past.' In this case, *ssb* is an "argument" for the command's usage,
not the command itself or its definition.

Again, in the Python implementation, this sense of "argument" is passed to the `CommandClass.do()` method. If you're unsure what that means, the important
point to take away is that **arguments passed to commands by users in chat DO NOT permanently alter the functionality of the command itself.** They only affect
what it does in that particular instance.

### JSON Specification for Commands

In the example JSON provided above, the use of `:command1` etc. is a shorthand for the following syntax:

```json
// :command
["CommandClass", "command_name", "param1", "param2",...]
```

In JSON terms, this is an array of strings (and in certain cases, other data types) whereby the first string specifies the particular type of command you are defining.
The second string specifies the name of the command, and the following parameters will depend on the type of command you are defining and what you want it to do.
The different types of commands are defind in the following section, but first let's go over:

### Inner Commands

When we get to the specific command types below, you will occasionally see the use of `:inner_command` within the list of parameters. Again, this is a shorthand
for a particular JSON specification, but when defining inner commands this way, the syntax is slightly different and can take one of three forms.

#### CommandName Strings

Inner commands can be defined by simply passing the name of the other command as a string:

```json
// :command
["OuterCommandClass", "outer_command_name", 
  "outer_command_params",... 
  // :inner_command
  "inner_command_name"
]
```

**Note**: In this case, do make sure that `inner_command_name` is actually defined somewhere in your `bot_settings.json` file, otherwise `stream.py` 
will throw an error when trying to initialize your bot.

#### Command/Argument Arrays

Inner commands can be invoked by an outer command with a certain set of arguments already specified, like so:

```json
// :command
["OuterCommandClass", "outer_command_name", 
  "outer_command_params",...
  // :inner_command
  [
    "inner_command_name", 
    "inner_command_arg1", 
    "inner_command_arg2",...
  ]
]
```

**Note**: In addition to ensuring the `inner_command_name` is actually defined, notice that the inner command and its arguments are contained
within their own array. A common mistake I make when creating command sets is forgetting to wrap them in this inner array, which will usually
cause `stream.py` to throw an error and almost certainly won't give you the right functionality.

#### Anonymous Command Definitions

The syntax for this type of inner command is exactly the same as the top level list of commands you are defining for your `public` and `restricted` arrays,
the only difference being that they are not given a name. Consequently, these commands can't be invoked on their own and in a sense
they "belong" to the outer command they are a parameter for.

```json
// :command
["OuterCommandClass", "outer_command_name",
  "outer_command_params",...
  // :inner_command
  [
    "InnerCommandClass", 
    "inner_parameter1", 
    "inner_parameter2",...
  ]
]
```

**Note:** Unlike the "Command/Argument Array" syntax mentioned above, you can't specify arguments for this type of inner command, only its initialization parameters.
If you think about it, this makes sense because this "anonymous" command can't be invoked anywhere else, so its functionality can't depend on receiving certain arguments
when invoked.

---

## CommandClasses

I've used the term "`CommandClass`" throughout this document to refer to (in Python terms) subclasses of the base class `Comand` as defined in `commands.py` in the source code. 
The names of these subclasses (with the CamelCase convention, as listed below) are the literal strings you use to specify what type of command you are creating.

The following subclasses define the types of commands you can create, but as you will see, their combined usage can get quite complicated and powerful
because of the way they can be joined together and used to invoke each other. We'll start with the simplest examples and work our way up.


### TextCommand

```json
["TextCommand", "name", 
  "Text for your bot to send to the chat"
]
```

The simplest use case, this type of command simply causes your bot to say something to the chat whenever it is invoked.

### FormatCommand

```json
["FormatCommand", "name", 
  "Formatted text with {variable_interploation}", 
  (optional) {"inner_keyword": "some value",...}
]
```

Similar to the `TextCommand` class, the `FormatCommand` supports a type of variable interpolation based on the specification in Python's
`str.format()` method. The set of variables passed to this formatting method come from the bot's state variables
and they can be interpolated into the string by enclosing the variable name in curly brackets like this: `"{variable_name}"`

An optional final parameter allows for the use of a JSON object that provides additional keyword/value pairs to be interpolated as variables.
There's not really a reason to do this explicitly in your `bot_settings.json` file, but if used in conjuction with the `GetCommand` 
it allows you to take the response from an external API and format it using values from the JSON that is returned from the GET request.

**Note:** These variable interpolations can also support key or index arguments: `{variable[subkey]}`

For example, if you have a state variable called `list` that looks like `["One", "Two", "Three"]` then given the formatting parameter `"{list[0]}"`
your bot will send the message *One* to the chat.

Python's string formatting method also supports some additional formatting arguments as specified [here.](https://docs.python.org/3/library/string.html#format-specification-mini-language)
This allows for padding, alignment, and some numeric conversion. Bear in mind that in the source code's implementation,
the `str.format()` method is called with the state variables passed as named keywords, so the `FormatCommand` class itself doesn't
support the use of positional arguments in this expression syntax.

### StateCommand

```json
["StateCommand", "name", 
  (optional) "key"
]
```

The `StateCommand` sets the value of some state variable to the arguments passed to it. 

By default, the name of the command is used as the name for the state variable, but the optional third parameter can be specified 
so that the command's name is different from the name of the state variable itself.

**Note:** The arguments passed to this command when invoked are concatenated into a single string, so for example,
the message *!variable1 argument1 argument2* 

### SubStateCommand

```json
["SubStateCommand", "name", 
  "variable_name", "variable_key"
]
```

Similar to the `StateCommand`, the `SubStateCommand` can be used to set the value of some key within an object, or index of an array
that is stored as a state variable. 

**Note:** Arrays stored as state variables are zero-indexed, so to return the first value of an array
the number `0` should be used as the final parameter.

**Note:** When setting a value by index within an array, you cannot "create" a new array element by accessing an array index that doesn't exist yet.
You can, however, set the value of arbitrary keys on a state variable that is defined as a JSON object, should you wish to do so.

**Note:** there is no `SubSubStateCommand` class, so there is inherently a depth limit for key/index values that can be accessed
by using any command. Nothing prevents you from storing state variables with more complex nesting, but you should keep this limitation in mind
when creating state variables.

### JsonCommand

```json
["JsonCommand", "name", 
  {"some_key": "some value that can be {variable_interpolated}",...}
]
```

The `JsonCommand` class takes a third parameter that should be a JSON object rather than a string. The intended use for this
`CommandClass` is to be able to create a JSON formatted string that can then be passed to a `PostCommand` to make up the body of its POST request.

The value for each key within the JSON object can also be interpolated with state variables, just like the FormatCommand class.
This formatting is performed recursively so strings within nested objects will also be interpolated.


### PostCommand

```json
["PostCommand", "name", 
  "url"
]
```

The PostCommand is used to send a POST request to an API endpoint specified by the URL parameter.

**Note:** When the `PostCommand` is invoked in chat, the arguments passed to it must form a valid JSON serializable string, which is treated as
the body of the POST request. Because this is error prone and unwieldy, it's recommended that the `PostCommand` be used in conjunction 
with other commands, typically a `JsonCommand` in order to provide a properly fromatted JSON serializable string.

### GetCommand

```json
["GetCommand", "name", 
  "url"
]
```

Similar to the `PostCommand`, the `GetCommand` sends a GET request to an API endpoint specified by the URL parameter and sends the response to chat.

Because most web APIs return data as serialized JSON, it is generally more useful to chain this command with other commands, such as the `FormatCommand`,
similar to the use of `PostCommand` in reverse.

### AliasCommand

```json
["AliasCommand", "name", 
  "other_command", "arg1", "arg2"...
]
```

The `AliasCommand` is used to invoke some other command with a specified set of arguments passed to that other command. 
They are useful for creating a shorthand for a common command/argument pairs such as typing *!ssb* to invoke *!game Super Smash Bros.*

**Note:** The parameter for the other command to be invoked must be a string and refer to a defined command.
The "Command/Argument Arrays" and "Anonymous Command Definitions" mentioned in the previous section don't work here.

### ChainCommand

```json
["ChainCommand", "name", 
  "output_command", "input_command"
]
```

The ChainCommand is used to pass the "output" (i.e.: any message that would be sent to chat) of one command to the "input" of another
(i.e.: the arguments that would be passed if the command were invoked in chat).

This command is particularly powerful when combined with `SequenceCommands` and `OptionCommands`, but another common use case would be
to chain the output of a `JsonCommand` to the input of a PostCommand in order to, for example, use a set of state variables to make up
the body of a POST request for an external API.

**Note:** The parameters for the output and input commands must be strings referring to defined commands.
The "Command/Argument Arrays" and "Anonymous Command Definitions" mentioned in the previous section don't work here.

### SequenceCommand

```json
["SequenceCommand", "name", 
  :inner_command1, 
  :inner_command2...
]
```

The `SequenceCommand` will invoke a series of other commands, passing any arguments it receives to each command in the series it invokes.

**Note:** Any inner commands defined here with a specified set of arguments will have those arguments passed to it **before** the arguments
from the chat when the SequenceCommand is invoked.

So for example, if I define a `SequenceCommand` as follows:

```json
["SequenceCommand", "sequence", 
  ["inner_command", "inner_arg1", "inner_arg2"]
]
```

I could invoke it in chat with the message: 

*!sequence outer_arg1 outer_arg2*

The inner command will then be invoked as if the following message was sent in chat: 

*!inner_command inner_arg1 inner_arg2 outer_arg1 outer_arg2*

### OptionCommand

```json
["OptionCommand", "name", 
  ["option1", :inner_command1], 
  ["option2", :inner_command2]...
]
```

The `OptionCommand` takes some argument and uses it to determine which from a series of other commands to invoke, 
passing all additional arguments after the first one to that inner command. 

**Note:** The same rules apply for passing outer vs. inner arguments as with **SequenceCommand** except that here,
the first argument passed to the `OptionCommand` is not included.

So for example, I could define an `OptionCommand` as follows:

```json
["OptionCommand", "option",
  ["choice1", ["inner_command", "inner_arg1"]],
  ["choice2", ["inner_command", "inner_arg2"]]
]
```

It could then be invoked in chat by sending:

*!option choice1 outer_arg*

Which would be the same as sending the message:

*!inner_command inner_arg1 outer_arg*

**Note:** Each of the option parameters is itself an array, so when used in conjunction with "Command/Argument Arrays" and "Anonymous Command Definitions"
each option will be an array with a string for each choice representing the first item and then an additional array as the second item.
A common pitfall is forgetting to enclose an anonymous command, for example, in its own array.
