# Restaurant Suggestion and Voting Protocol

**VERSION 0.1**

The Restaurant Suggestion and Voting Protocol (RSVP) allows members of an organization (an *electorate*) to nominate and vote on dining locations for a lunch trip. Deliberately over-engineered and has a user-facing syntax of HTTP. Solve the pesky human problem of your coworkers being indecisive or apathetic about sustenance by injecting *technology*!

## Definitions

### Entities

* **Server** - An automated service that organizes elections and communicates with citizens (over RSVP) to produce election results.
* **Citizen** - A human entity that participates in elections. Analogous to a "client" in other protocol terminology.
* **Electorate** - A group of citizens that participate in elections. Each electorate has their own server.
* **Location** - A representation of a real-world eatery, typically stored by the server on a database. Consists of:
	* **Name** - An identifier used in the protocol (though the server may allow aliases).
	* **Metadata** - Plugin- or implementation-specific key-value pairs to describe the location further.

### Elections

* **Election** - A periodic event in which the electorate selects a location that should be attended for lunch. The exact details of an election are configurable; see Election System.
* **Suggest** - A command from a citizen to indicate a location should be available for voting.
* **Candidate** - A location that has been suggested.
* **Vote** - A command from a citizen to indicate their opinion about a candidate.
* **Results** - The outcome of an election.

### Communication

* **Command** - An instruction given by a citizen to the server. These include suggesting candidates and voting.
* **Reply** - A response to given by the server in response to a citizen's command.
* **Decree** - Information given by the server to the electorate.
* **Registration** - An action a citizen can take to

### Customization

* **Plugin** - A piece of coded functionality that a server can use to alter its behavior. The format and installation of these plugins are implementation-defined, but some kinds of customization are explicitly defined by this specification.
* **Superdelegate** - A plugin that suggests locations. Any number, including none, may be installed on a server.
* **Vetoer** - A plugin that can reject suggestions, from both superdelegates and citizens. Any number, including none, may be installed on a server.
* **Election System (ES)** - A plugin that defines how an election is carried out, such as how votes are counted, and how the results are presented. A server must always have an ES installed, though an implementation may provide a default implementation in absence of a suitable plugin.

## Protocol vs. Implementation

This protocol does not specify how citizens communicate with the server, only the content of that communication. While conforming implementations must allow the textual representations specified here, they may also include other means of communication, such as a graphical interface.

## Commands

Commands are issued in the form:

<pre>
(sentinel) (verb) [path value]
[option_name: option value]
[option_name: option value]
[...]
</pre>

* **Sentinel** - A keyword or phrase introducing the command.
* **Verb** - The specific kind of command to issue.
* **Path** - An argument for the command, such as a location name.
* **Options** - Additional arguments for some commands.

For example,

<pre>
RSVP/0.1 VOTE Joe's
</pre>

or 

<pre>
RSVP METADATA Taco House
Kind: Mexican
TravelTime: 2
</pre>

### Sentinel

A sentinel is used to indicate the text being issued is relevant to RSVP. This is necessary in shared interactive environments, such as chat rooms, where not all text posted to the room needs to be taken as input by the server. All implementations should support sentinels, but depending on the use case they might not require them.

Sentinels take the form: `RSVP[/(major).(minor)]`.

The phrase `RSVP` is case-insensitive.

Version information is optional, but if it is specified, the server is required to validate that it supports the protocol version specified. For instance, the following are all valid sentinels for a server supporting all versions up to `1.1`:

`RSVP`  
`RSVP/0.1`  
`RSVP/1.0`  
`RSVP/1.1`  

If the version given is not supported by the server, the server will issue a `F1 Incompatible Version` reply and not carry out the command.

### Options



### Verbs

Verbs are case-insensitive.

In addition to the verbs specified here, implementations may define other verbs to be used, including as aliases of protocol-specified verbs.

#### SUGGEST

Issued by a citizen to indicate a location should be considered in the current election.

* **Path**: Required. The name of the location.
* **Options**: None protocol-defined. May include information about why the location is being proposed, or stipulations about its inclusion.

The exact result of a citizen suggesting a location is dependent on the ES, but typically it will add the suggested location to a list of candidates that can then be voted on. Because suggestions may be issued in private channels, the ES should decree when a new suggestion is added, or decree the full list at a regular interval.

*Examples:*

<pre>
RSVP/0.1 SUGGEST Joe's
</pre>

<pre>
RSVP/0.1 SUGGEST Joe's
Why: Group special, one day only
When: Before noon, they're usually crowded
</pre>

#### VOTE

#### RESCIND

#### REGISTER (citizen)

#### REGISTER (location)

#### METADATA

#### OPTIONS

## Replies

### S Success

### Ex Command Error

### Fx Server Failure

#### F
0 General Failure

#### F1 Incompatible Version

The server does not support the version specified by the command's sentinel.

For instance, if a server only supports version `1.0` and a citizen issues `RSVP/2.1 VOTE Joe's`, the server must reply with `F1`. This is the case even if the command could be issued with a `1.0` sentinel and be processed successfully; because the server does not support version `2.1`, it has no way of knowing whether the `1.0` processing of the command is correct when the citizen expects the `2.1` behavior.

## Decrees

Decrees are issued by servers in the form:

<pre>
(sentinel) (decree_number) (decree_name)

[body]
</pre>

* **Sentinel** - A keyword or phrase introducing the decree (same as in commands; see above).
* **Decree number and name** - The specific kind of decree being given.
* **Body** - Human-readable information about the decree, if needed.

For example,

<pre>
RSVP/0.1 111 Primary Begins

Today's primary has started. SUGGEST lunch locations.
The primary will close at 11:45 AM.
</pre>

### 0xx Plugin Reserved

### 1xx In Progress

### 2xx Success

### 3xx Abnormal

#### 300 Multiple Choices

More than one candidate received the highest number of votes in a general.

#### 301 Delay

The current phase of the election has been delayed by a commissioner.

### 4xx Failure

## Plugin Interface

## Future Development