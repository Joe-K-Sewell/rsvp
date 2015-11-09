# Restaurant Suggestion and Voting Protocol

**VERSION 0.1**

## Introduction

The Restaurant Suggestion and Voting Protocol (RSVP) allows members of an organization (an *electorate*) to nominate and vote on dining locations for a lunch trip. Deliberately over-engineered and has a user-facing syntax of HTTP. Solve the pesky human problem of your coworkers being indecisive or apathetic about sustenance by injecting *technology*!

## Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](http://tools.ietf.org/html/rfc2119).

### Entities

* **Server** - An automated service that organizes elections and communicates with citizens (over RSVP) to produce election results.
	* An **implementation** - Unless otherwise qualified, refers to an implementation of a Server, including how its behavior is affected by installed Plugins.
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
* **Reply** - A response to given by the server to a citizen's command.
* **Decree** - Information given by the server to the electorate.
* **Registration** - An action a citizen can take to tell the server it should store information on the citizen itself, or a location. Implementations may have registration be optional, and automatically detect new citizens and/or locations.

### Customization

* **Plugin** - A piece of coded functionality that a server can use to alter its behavior. The format and installation of these plugins are implementation-defined, but some kinds of customization are explicitly defined by this specification and SHOULD be implemented.
* **Superdelegate** - A plugin that suggests locations. Zero or more may be installed on a server.
* **Vetoer** - A plugin that can reject suggestions, from both superdelegates and citizens. Zero or more may be installed on a server.
* **Election System (ES)** - A plugin that defines how an election is carried out, such as how votes are counted, and how the results are presented. Each election MUST have an ES, though an implementation may provide a default ES in absence of a suitable Plugin.

### Syntax

In this document, the following symbols are used when describing a prototypical syntactical entity:

* `(` and `)` surround a term that is used as a placeholder.
* `[` and `]` surround a term that is optional.
* `[...]` indicates "and so on".
* All other terms that are not surrounded by `(` and `)` indicate a case-insensitive literal character sequence.

For instance, `RSVP[/(major).(minor)]` defines a structure that:

* Begins with the literal characters `RSVP`
* Is optionally followed by another term:
	* Which begins with the literal `/`
	* Followed by a placeholder called `major` which is defined after the definition
	* Followed by a literal `.`
	* Followed by a placeholder called `minor` which is defined after the definition

## Versioning

### Specification

This specification is a public API that adheres to [Semantic Versioning](http://semver.org/). This means:

* The version of this specification consists of a major and minor version.
* The major version delimits breaking changes.
* The minor version delimits changes which are compatible with prior minor versions of the same major version.

For instance, consider a command sent to versions `1.0`, `1.1`, and `2.0`. Version `1.1` will do everything `1.0` did, although additional effects may also happen (provided these do not violate `1.0`). Version `2.0` can behave totally differently than both `1.0` and `1.1`.

### Implementation

A server:

* SHOULD version itself separately from this specification, to enable feature development and enhancements unrelated to RSVP proper (such as plugin capabilities).
* MAY support multiple specification versions.
* SHOULD document all specification versions it supports.
* MAY only explicitly implement one minor version of a given major specification version, and through that implementation support all prior minor versions of that major version.
* MUST decree using only the latest supported specification version.

Servers are REQUIRED to select versions to service commands by the algorithm listed in the [Version Selection process](#processes-version-selection).

## Implementation Considerations

This protocol does not specify how citizens communicate with the server, only the content of that communication. While conforming implementations SHOULD allow the textual representations specified here, they may also include other means of communication, such as a graphical interface.

### Citizen identification

The implementation MUST have a way to distinguish and verify citizens, including associating every command issued with a particular citizen. If the underlying protocol already has a user system, it may already be sufficient for this purpose. If no such method exists, options may be used on commands to authenticate citizen commands.

### Public vs. Private Channels

If the underlying protocol allows direct communication between a citizen and the server, this method is called a **private channel**. Other methods, which allow other citizens to observe the communication, are **public channels**.

The presence of private channels is not a prerequisite for using a given underlying protocol, though it presents challenges to anonymous voting. If private channels are present, the implementation SHOULD allow commands to be issued over them. If a command is received over such a channel, the reply MUST also be on a private channel.

Decrees SHOULD be made on public channels, if available. Otherwise, they SHOULD be transmitted simultaneously to all participating citizens.

## Commands

A command is an instruction given by a citizen to the server. The server will react appropriately, and transmit a Reply to the citizen to inform that citizen of the result.

Commands are issued in the form:

	(sentinel) (verb) [(path value)]
	[(option_name): (option value)]
	[(option_name): (option value)]
	[...]

* **Sentinel** - A key phrase introducing the command.
* **Verb** - The specific kind of command to issue.
* **Path** - An argument for the command, such as a location name.
* **Option name** and **option value** - Key-value pairs that can be used with some commands.

### Sentinel

A sentinel is used to indicate the text being issued is relevant to RSVP. This is necessary in public channels, such as chat rooms, where not all text posted to the room needs to be taken as input by the server (i.e., some text posted to that medium may not have any bearing on RSVP at all). All implementations SHOULD support sentinels, but depending on the use case they might not require them.

Sentinels take the form:

	RSVP[/(major).(minor)]

where `major` and `minor` correspond to the major and minor version number of the protocol version the command expects.

Version information is OPTIONAL for sentinels specified in commands, but if it is specified, the server is REQUIRED to validate that it supports that version. For instance, the following are all valid sentinels for a server supporting both `1.0` and `1.1`:

`RSVP`  
`RSVP/1.0`  
`RSVP/1.1`  

If the version given is not supported by the server, the server MUST reply `505 RSVP Version Not Supported` and MUST NOT carry out the command.

### Path

Some verbs take an argument, called a "path". Depending on the implementation, this value MAY:

* Be case-sensitive
* Be expressed in a hierarchical structure with elements separated by `/`
* Contain inner whitespace

However, the path MUST NOT contain the line-break character, nor treat leading or trailing whitespace as significant.

### Options

Some verbs take additional arguments, expressed as key-value pairs, called "options". Implementations MAY also define how options are used. For instance, though the `SUGGEST` verb has no protocol-defined options, a Plugin might care about why a citizen chose that location for future use, and so a `Why` option can be provided:

	RSVP/0.1 SUGGEST Big Dipper
	Why: Group special, one day only

### Verbs

In addition to the verbs specified here, implementations may define other verbs to be used, including as aliases of protocol-specified verbs.

#### SUGGEST

Issued by a citizen to indicate a location should be considered in the current election.

* **Path**: Required. The name of the location.
* **Options**: None protocol-defined.

The exact result of a citizen suggesting a location is dependent on the ES, but typically it will add the suggested location to a list of candidates that can then be voted on.

Because suggestions, like all commands, may be issued in private channels, the ES SHOULD either decree when a new suggestion is added and/or decree the full list at a regular interval.

#### VOTE

Issued by a citizen to voice their opinion on a location in the current election.

* **Path**: Required. The name of the location.
* **Options**: None protocol-defined.

The exact result of voting on a location is dependent on the ES, but typically it will increment a counter for the suggested location. The ES can choose to restrict the amount of votes a citizen can make. If the ES provides multiple kinds of votes, the kind can be selected using an implementation option.

#### RESCIND

Issued by a citizen to retract a prior VOTE.

* **Path**: Implementation-dependent.
* **Options**: None protocol-defined.

The ES SHOULD reverse all effects made by the prior VOTE. If there can be multiple VOTEs, the ES MAY either reverse all VOTEs with a single RESCIND, or require the vote be specified by the path and/or options.

#### REGISTER

Issued by a citizen to inform the server of the existence of an entity: either the citizen themself, or a location.

* **Path**:
	* If registering the citizen, do not include.
	* If registering a location, the location name.
* **Options**:
	* If registering the citizen, none protocol-defined.
	* If registering a location, metadata to include with the location.

Implementations MAY automatically register citizens and locations as they are used, instead of requiring the use of this verb.

#### METADATA

Issued by a citizen to update the metadata of a location.

* **Path**: Required. The name of the location.
* **Options**: The metadata key-value pairs to set.
	* If the option value is empty, the associated metadata will be removed.

#### LIST

Issued by a citizen to list locations, the metadata of a location, or citizens.

* **Path**: Optional. The name of a location or the literal `citizens`.
	* If omitted, the reply contains a list of location names.
		* If the *All* option is `true`, this list contains all registered locations.
		* Otherwise, this list contains the candidates in the current election.
	* If a location name, the reply contains the metadata for the given location.
		* The *All* option has no effect on this mode.
	* If `citizens`, the reply contains a list of citizen names.
		* If the *All* option is `false`, this list contains the citizens who have suggested or voted in the current election.
		* Otherwise, this list contains all registered citizens.
* **Options**:
	* *All*: Optional. If specified, `true` or `false`. See above for behavior.

The items in all lists returned are separated by line breaks.

## Replies

### 0xx Plugin-reserved

### 1xx Pending Server Action

### 2xx Success

### 3xx Pending Citizen Action

### 4xx Citizen Error

### 5xx Server Error

#### 505 RSVP Version Not Supported

The server does not support the version specified by the command's sentinel.

For instance, if a server only supports version `1.0` and a citizen issues `RSVP/1.1 VOTE Joe's`, the server MUST reply with `505`. This is the case even if the command could be issued with a `1.0` sentinel and be processed successfully; because the server does not support version `1.1`, it has no way of knowing whether the `1.0` processing of the command is correct when the citizen expects the `1.1` behavior (which, per Semantic Versioning, may have added behavior to the verb).

## Decrees

Decrees are issued by servers in the form:

	(sentinel) Decree (decree_number) (decree_name)
	
	[(body)]

* **Sentinel** - A keyword or phrase introducing the decree (same as in commands; see above).
* **Decree number** - A canonical number corresponding to the kind of decree issued. Expressed in a series of natural numbers, delimited by `.` - e.g., `3.0`.
* **Decree name** - A human-readable description of the decree.
* **Body** - Human-readable information contained in the decree. May have further line-breaks.

### 0 Plugin-reserved

### 1 In Progress

Issued to indicate a process has been started and continues.

#### 1.0 Election Status

Issued by the ES to report periodic results, such as the current candidates.

	RSVP/0.1 Decree 1.0 Election Status
	
	Current suggestions:
	Big Dipper, Cookie Factory, Joe's, Tex-Mex House 

	5 minutes remain to SUGGEST.

#### 1.1 Election Begins

The ES has begun an election. Can be issued alongside `1.2` and/or `1.3`.

	RSVP/0.1 Decree 1.1 Election Begins
	
	Today's election has opened.

#### 1.2 Suggestion Begins

The ES is now accepting suggestions for an election.

	RSVP/0.1 Decree 1.2 Suggestion Begins
	
	Today's primary has started. SUGGEST lunch locations.
	The primary will close at 11:45 AM.

#### 1.3 Voting Begins

The ES is now accepting votes on candidates.

	RSVP/0.1 Decree 1.3 Voting Begins
	
	Today's general election has started. VOTE on candidates.
	The voting will close at 12:00 Noon.

### 2 Success

#### 2.1 Election Results

The ES has completed an election and has results.

	RSVP/0.1 Decree 2.1 Election Results
	
	Joe's (3 votes)
	Tex-Mex House (2 votes)
	Big Dipper (2 votes)
	Cookie Factory (0 votes)

#### 2.2 Suggestion Closed

The ES has stopped accepting suggestions.

#### 2.3 Voting Closed

The ES has stopped accepting votes.

### 3 Abnormal

### 4 Failure

# Processes

## Command Handling

<a name="processes-version-selection"></a>
### Version Selection

Servers may implement multiple versions of this protocol. To ensure that commands are processed by the semantics intended by the issuing citizen, this section describes an algorithm that finds the best fit for a given command's version. If the specified version is not supported, this algorithm also ensures the error reply contains relevant version information for the citizen to consider. Servers MUST implement this algorithm to ensure consistent behavior.

Consider the protocol versions a server supports. They can be thought of being stored in a mapping, `V`, of integers (the major versions) to sets of integers (the minor versions). When a server receives a command:

1. If the command did not specify a version:
	1. Let `a` be the greatest key in `V`.  
	2. Let `b` be the greatest value in `V[a]`.
	3. Implementation `a.b` services the command, and this version number is used on replies to the command. **Processing stops here.**
2. Let `x.y` be the specified version.
3. If `x` is not a key in `V`:
	1. Let `a` be the greatest key in `V`.  
	2. Let `b` be the greatest value in `V[a]`.
	3. The command is rejected with a reply of `505 RSVP Version Not Supported`. The reply indicates version `a.b`. **Processing stops here.**
4. Let `S` be the subset of `V[x]` where each element `s` satisfies `y <= s`.
5. If `S` is empty:
	1. Let `b` be the greatest value in `V[x]`.
	2. The command is rejected with a reply of `505 RSVP Version Not Supported`. The reply indicates version `x.b`. **Processing stops here.**
6. Let `m` be the the minimum element of `S`.
7. Implementation `x.m` services the command, and this version number is used on replies to the command. **Processing stops here.**

*Example:* For example, an "RSVPico" server implements versions `1.0`, `1.2`, and `2.0`. Thus,

	V = {1 -> {0, 2}, 2 -> {0}}

With the aforementioned example server:

* Citizens can send commands with version `1.0`. These commands will be fulfilled by the server's implementation of `1.0`. Replies will indicate version `1.0`. 
* Citizens can also send commands with either version `1.1` or `1.2`. These commands will be fulfilled by the server's implementation of `1.2`. Replies will indicate version `1.2`.
* Citizens can also send commands with version `2.0`.  These commands will be fulfilled by the server's implementation of `2.0`. Replies will use version `2.0`.
* Citizens can also send commands with no version specified.  These commands will be fulfilled by the server's implementation of `2.0`. Replies will use version `2.0`.
* Citizens can not successfully send commands with version `1.5`. These commands will not be processed. Replies will be `505 RSVP Version Not Supported` and use version `1.2`.
* Citizens can not successfully send commands with version `0.1` nor `2.1` nor `3.0`. These commands will not be processed. Replies will be `505 RSVP Version Not Supported` and use version `2.0`.