# Restaurant Suggestion and Voting Protocol

**VERSION 0.1**

## Introduction

The Restaurant Suggestion and Voting Protocol (RSVP) allows members of an organization (an *electorate*) to nominate and vote on dining locations for a lunch trip. Deliberately over-engineered and has a user-facing syntax of HTTP. Solve the pesky human problem of your coworkers being indecisive or apathetic about sustenance by injecting *technology*!

Though a parody of a client-server model, this specification does not describe the interaction of two automated components. Rather, it defines the interaction of a usually-human component (a *citizen*) with an automated system (a *service*).

## Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](http://tools.ietf.org/html/rfc2119).

### Entities

* **Service** - An automation that organizes elections and communicates with citizens (over RSVP) to produce election results.
	* An **implementation** - Unless otherwise qualified, refers to an implementation of a service, including how its behavior is affected by installed Plugins.
* **Citizen** - A human entity that participates in elections.
* **Electorate** - A group of citizens that participate in elections. Each electorate has their own service.
* **Location** - A representation of a real-world eatery, typically stored by the service on a database. Consists of:
	* **Name** - An identifier used in the protocol (though the service may allow aliases).
	* **Metadata** - Plugin- or implementation-specific key-value pairs to describe the location further.

### Elections

* **Election** - A periodic event in which the electorate selects a location that should be attended for lunch. The exact details of an election are configurable; see Election System.
* **Suggest** - A command from a citizen to indicate a location should be available for voting.
* **Candidate** - A location that has been suggested.
* **Vote** - A command from a citizen to indicate their opinion about a candidate.
* **Results** - The outcome of an election.

### Communication

* **Command** - An instruction given by a citizen to the service. These include suggesting candidates and voting.
* **Reply** - A response to given by the service to a citizen's command.
* **Decree** - Information given by the service to the electorate.
* **Registration** - An action a citizen can take to tell the service it should store information on the citizen itself, or a location. Implementations may have registration be optional, and automatically detect new citizens and/or locations.

### Customization

* **Plugin** - A piece of coded functionality that a service can use to alter its behavior. The format and installation of these plugins are implementation-defined, but some kinds of customization are explicitly defined by this specification and SHOULD be implemented.
* **Superdelegate** - A plugin that suggests locations. Zero or more may be installed on a service.
* **Vetoer** - A plugin that can reject suggestions, from both superdelegates and citizens. Zero or more may be installed on a service.
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

A service:

* SHOULD version itself separately from this specification, to enable feature development and enhancements unrelated to RSVP proper (such as plugin capabilities).
* MAY support multiple specification versions.
* SHOULD document all specification versions it supports.
* MAY only explicitly implement one minor version of a given major specification version, and through that implementation support all prior minor versions of that major version.
* MUST decree using only the latest supported specification version.

Services are REQUIRED to select versions to service commands by the algorithm listed in the [Version Selection process](#processes-version-selection).

## Implementation Considerations

This protocol does not specify how citizens communicate with the service, only the content of that communication. While conforming implementations SHOULD allow the textual representations specified here, they may also include other means of communication, such as a graphical interface.

### Citizen identification

The implementation MUST have a way to distinguish and verify citizens, including associating every command issued with a particular citizen. If the underlying protocol already has a user system, it may already be sufficient for this purpose. If no such method exists, options may be used on commands to authenticate citizen commands.

### Public vs. Private Channels

If the underlying protocol allows direct communication between a citizen and the service, this method is called a **private channel**. Other methods, which allow other citizens to observe the communication, are **public channels**.

The presence of private channels is not a prerequisite for using a given underlying protocol, though it presents challenges to anonymous voting. If private channels are present, the implementation SHOULD allow commands to be issued over them. If a command is received over such a channel, the reply MUST also be on a private channel.

Decrees SHOULD be made on public channels, if available. Otherwise, they SHOULD be transmitted simultaneously to all participating citizens.

## Commands

A command is an instruction given by a citizen to the service. The service will react appropriately, and transmit a Reply to the citizen to inform that citizen of the result.

Commands are issued in the form:

	(sentinel) (verb) [(path value)]
	[(option_name): (option value)]
	[(option_name): (option value)]
	[...]

* **Sentinel** - A key phrase introducing the command.
* **Verb** - The specific kind of command to issue.
* **Path** - An argument for the command, such as a location name.
* **Option name** and **option value** - Key-value pairs that can be used with some commands.

Note that the order of the elements in the first line is different than that of an HTTP request: the sentinel comes first.

### Sentinel

A sentinel is used to indicate the text being issued is relevant to RSVP. This is necessary in public channels, such as chat rooms, where not all text posted to the room needs to be taken as input by the service (i.e., some text posted to that medium may not have any bearing on RSVP at all). All implementations SHOULD support sentinels, but depending on the use case they might not require them.

Sentinels take the form:

	RSVP[/(major).(minor)]

where `major` and `minor` correspond to the major and minor version number of the protocol version the command expects.

Version information is OPTIONAL for sentinels specified in commands, but if it is specified, the service is REQUIRED to validate that it supports that version. 

For instance, the following are all valid sentinels for a service supporting both `1.0` and `1.1`:

`RSVP`  
`RSVP/1.0`  
`RSVP/1.1`  

If the version given is not supported by the service, the service MUST reply `INVALID VERSION` and MUST NOT carry out the command. 

For more information, see the [Version Selection process](#processes-version-selection).

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

Below, verbs are specified. Included with each definition are:

* **Path**: the value specified in the command's path when using this verb.
* **Options**: supported options when using this verb. Implementations may define additional options in both verb-dependent and -independent manners.
* **Replies**: the mnemonics of replies the service may use in response to the command. In addition to these, all verbs may trigger the following:
	* Any verb that could trigger a form of `ACCEPT` MAY also attach the flag `NEW-CITIZEN` if the citizen was not previously registered, but the service automatically registered the citizen while processing this command. 
	* `REFUSE CITIZEN` if the service restricts the use of the command to certain citizens, and the issuing citizen is not permitted.
	* `INVALID` if the command could not be interpreted by the service.
	* `INVALID VERSION` if the service does not support the version specified by the command.
	* `FAILURE` if the service encountered an internal error when processing the command.

In addition to the verbs specified here, implementations may define other verbs to be used, including as aliases of protocol-specified verbs.

#### SUGGEST

Issued by a citizen to indicate a location should be considered in the current election.

* **Path**: Required. The name of the location.
* **Options**: None protocol-defined.
* **Replies**:
	* `ACCEPT` if the location was suggested successfully. Possible flags:
		* `NEW-LOCATION` if the location was unrecognized by the service, but the service automatically registered it and allowed the suggestion to complete.
	* `REFUSE` if the suggestion was not taken into account. Reasons might include the location being vetoed, or having already been suggested.  
	* `REFUSE TIME` if the service is not accepting suggestions at this time.

The exact result of a citizen suggesting a location is dependent on the ES, but typically it will add the suggested location to a list of candidates that can then be voted on.

Because suggestions, like all commands, may be issued in private channels, the ES SHOULD either decree when a new suggestion is added and/or decree the full list at a regular interval.

#### VOTE

Issued by a citizen to voice their opinion on a location in the current election.

* **Path**: Required. The name of the location.
* **Options**: None protocol-defined.
* **Replies**:
	* `ACCEPT` if the vote was counted successfully. Possible flags:
		* `NEW-CANDIDATE` if the location hadn't been suggested, but the service automatically suggested it and allowed the vote to be counted.
		* `NEW-LOCATION` if the location was unrecognized by the service, but the service automatically registered it and allowed the vote to complete.
	* `REFUSE` if the vote was not taken into account. Reasons might include the citizen having already used all of their votes, or the location not being suggested.
	* `REFUSE TIME` if the service is not accepting votes at this time.

The exact result of voting on a location is dependent on the ES, but typically it will increment a counter for the suggested location. The ES can choose to restrict the amount of votes a citizen can make. If the ES provides multiple kinds of votes, the kind can be selected using an implementation option.

#### RESCIND

Issued by a citizen to retract a prior VOTE.

* **Path**: Implementation-dependent.
* **Options**: None protocol-defined.
* **Replies**:
	* `ACCEPT` if the vote was rescinded successfully.
	* `REFUSE` if the vote could not be rescinded. Reasons might include the specified vote having not been made. Possible flags:
		* `TIME` if the service is not accepting votes at this time, so no there are no votes to rescind.

The ES SHOULD reverse all effects made by the prior VOTE. If there can be multiple VOTEs, the ES MAY either reverse all VOTEs with a single RESCIND, or require the vote be specified by the path and/or options.

#### REGISTER

Issued by a citizen to inform the service of the existence of an entity: either the citizen themself, or a location.

* If registering a citizen:
	* **Path**: Omitted.
	* **Options**: None protocol-defined.
	* **Replies**:
		* `ACCEPT NEW-CITIZEN` if the citizen was registered.
		* `REFUSE` if the citizen could not be registered. Reasons might include the citizen already being registered.
* If registering a location:
	* **Path**: The location name.
	* **Options**: The metadata to include with the location.
	* **Replies**:
		* `ACCEPT NEW-LOCATION` if the location was registered.
		* `REFUSE` if the location could not be registered. Reasons might include the location already being registered. Possible flags:
			* `TIME` if the service is not allowing locations to be registered at this time.

Implementations MAY automatically register citizens and locations as they are used, instead of requiring the use of this verb.

#### METADATA

Issued by a citizen to update the metadata of a location.

* **Path**: Required. The name of the location.
* **Options**: The metadata key-value pairs to set.
	* If the option value is empty, the associated metadata will be removed.
* **Replies**:
	* `ACCEPT` if the metadata was updated.
	* `REFUSE` if the metadata could not be updated. Reasons might include the location not having been registered. Possible flags:
		* `TIME` if the service is not allowing metadata to be modified at this time.

#### LIST

Issued by a citizen to list all locations, candidate locations, the metadata of a location, or citizens.

* If listing all locations:
	* **Path**: Optional. The literal `all`.
	* **Options**: None protocol-defined.
	* **Replies**:
		* `ACCEPT`, the body MUST contain a newline-separated list of all location names.
* If listing candidate locations:
	* **Path**: Required. The literal `candidates`.
	* **Options**: None protocol-defined.
	* **Replies**:
		* `ACCEPT`, the body MUST contain a newline-separated list of location names that have been suggested.
		* `REJECT TIME` if there is no election currently in progress.
* If listing the metadata of a location:
	* **Path**: Required. The name of a location.
	* **Options**: None protocol-defined.
	* **Replies**:
		* `ACCEPT`, the body MUST contain a newline-separated list of metadata key-value pairs in the form `key: value`.
		* `REFUSE` if the location is not registered.
* If listing all citizens:
	* **Path**: Required. The literal `citizens`.
	* **Options**: None protocol-defined.
	* **Replies**:
		* `ACCEPT`, the body MUST contain a newline-separated list of citizen names.

In reply bodies, except when listing the metadata of a location, additional information MAY be specified with each line item, enclosed in parentheses (the literal characters `(` and `)`).

For instance, the command `RSVP/0.1 LIST candidates` can indicate in its reply, if the ES permits it, the number of votes each location has received:

	ACCEPT RSVP/0.1
	
	Joe's (2 votes)
	Big Dipper (1 vote)
	Thai Buffet (1 vote)
	Trout-tastic! (0 votes)

## Replies

After receiving and processing a command, the service issues a corresponding **reply**. If the command was received by a private channel, the reply MUST also be transmitted by private channel.

Commands are issued in the form:

	(reply mnemonic) [(reply flags)] (sentinel)
	
	[(body)]

* **Reply mnemonic** - A textual representation of the type of reply.
* **Reply flags** - Additional indicators for the reply, specified by each mnemonic.
* **Sentinel** - A key phrase indicating the version of the service logic that handled the command.
* **Body** - Additional human- (and occasionally machine-) readable text describing the result.

Note that the order of the elements in the first line is different than that of an HTTP response: the sentinel comes last.

### Sentinel

Unlike the sentinel of the command, a reply's sentinel is REQUIRED. This is so that the citizen can know under what protocol version semantics their command was handled.

Sentinels take the form:

	RSVP/(major).(minor)

where `major` and `minor` correspond to the major and minor version number of the protocol version that was used to service the command.

For information about which version that MUST be given, see the [Version Selection process](#processes-version-selection).

### Body

Some replies have additional information, such as the results of a `LIST` command. This information SHOULD be human-readable, and may contain line breaks.

### Mnemonics

#### ACCEPT

The command was received and processed successfully.

Flags:

* `NEW-CITIZEN`: The citizen who sent the command was not previously recognized by the service, but now is. This may be due to a `REGISTER` command, or the service may automatically register citizens when they send their first commands. 
* `NEW-LOCATION`: The location specified by the command was not previously recognized by the service, but now is. This may be due to a `REGISTER` command, or the service may automatically register locations when they are first mentioned by commands.
* `NEW-CANDIDATE`: The location specified by the command was not previously a candidate in the current election, but now is. This may be due to a `SUGGEST` command, or the service may automatically consider a `VOTE` for an un-suggested location to be an implicit suggestion.

#### REFUSE

The command was received and determined valid, but denied. The body SHOULD explain why.

Flags:

* `TIME`: The command cannot be issued at the current time. For instance, attempting to `VOTE` when there is currently no election in progress.
* `CITIZEN`: The issuing citizen is not authorized to do so. Note this specification does not explicitly make such restrictions, but implementations MAY do so.

#### INVALID

The command was received but could not be considered because it was malformed or not supported. The body MAY explain why.

Flags:

* `VERSION`: The version specified by the command is not supported by the service. See the [Version Selection process](#processes-version-selection).

#### FAILURE

The service encountered an unexpected error. The body MAY contain details about the failure.

This mnemonic has no flags.

## Decrees

Decrees are issued by service in the form:

	(decree_number) (decree_name) (sentinel)
	
	(body)

* **Sentinel** - A keyword or phrase introducing the decree.
* **Decree code** - A canonical three-digit number corresponding to the kind of decree issued.
* **Decree name** - A human-readable description of the decree.
* **Body** - Human-readable information contained in the decree. May have further line-breaks.

### Sentinel

Same as for replies. Note that the version specified is the version that generated the decree, which MUST always be the newest supported version.

### Body

All decrees SHOULD contain explanatory text or additional information. See each decree code/name for more information.

### Codes 0xx: Intermediate Results

Provides information about a currently-running action.

#### 000 Time Warning

Indicates that the time remaining for citizens to perform an action, such as vote, is limited. The body MUST indicate the time limit (either in time span remaining, or a defined time), and what action will be unavailable at that time.

#### 001 Running Totals

A list of the current candidates in the election. The body of the decree MUST be identical to that of a reply from a `LIST candidates` command.

### Codes 1xx: Begin

Indicates that the service is beginning some action, which may mean certain commands will now be permitted.

#### 100 Election Begins

An election has begun. The body SHOULD indicate when the election will end.

#### 101 Suggestions Open

Citizens can now use the `SUGGEST` command. The body SHOULD indicate when suggestions will be closed.

#### 102 Voting Open

Citizens can now use the `VOTE` command. The body SHOULD indicate when voting will be closed.

#### 110 Service Available

Announces that the service is available to receive commands.

### Codes 2xx: Complete

Indicates that the service has completed some action, which may mean certain commands are no longer permitted.

#### 200 Election Complete

The election has finished. The body MUST contain the results of the election, however the ES defines that.

#### 201 Suggestions Closed

Citizens can no longer use the `SUGGEST` command.

#### 202 Voting Closed

Citizens can no longer use the `VOTE` command.

#### 210 Service Closing

Announces that the service is no longer receiving commands, due to an expected shutdown.

### Codes 3xx: Information Needed

Indicates that the electorate needs to provide further information in order for the service to complete some action.

#### 300 Multiple Choices

The election resulted in a tie. The body MUST contain the results of the election, however the ES defines that.

Based on the ES, the service MAY run another election (emitting a new `100` decree) to break the tie.

#### 301 Moved Permanently

#### 302 Found

#### 303 See Other

#### 304 Not Modified

### Codes 4xx: Electorate Error

Indicates that the service failed to complete some action because information or commands were not provided by the electorate.

#### 401 Unauthorized

#### 402 Payment Required

#### 403 Forbidden

#### 404 Not Found

#### 418 I'm a Teapot

### Codes 5xx: Service Error

Indicates that the service failed to complete some action, due to an internal issue.

# Processes

## Command Handling

<a name="processes-version-selection"></a>
### Version Selection

Services may implement multiple versions of this protocol. To ensure that commands are processed by the semantics intended by the issuing citizen, this section describes an algorithm that finds the best fit for a given command's version. If the specified version is not supported, this algorithm also ensures the error reply contains relevant version information for the citizen to consider. Services MUST implement this algorithm to ensure consistent behavior.

Consider the protocol versions a service supports. They can be thought of being stored in a mapping, `V`, of integers (the major versions) to sets of integers (the minor versions). When a service receives a command:

1. If the command did not specify a version:
	1. Let `a` be the greatest key in `V`.  
	2. Let `b` be the greatest value in `V[a]`.
	3. Implementation `a.b` services the command, and this version number is used on replies to the command. **Processing stops here.**
2. Let `x.y` be the specified version.
3. If `x` is not a key in `V`:
	1. Let `a` be the greatest key in `V`.  
	2. Let `b` be the greatest value in `V[a]`.
	3. The command is given an `INVALID VERSION` reply. The reply indicates version `a.b`. **Processing stops here.**
4. Let `S` be the subset of `V[x]` where each element `s` satisfies `y <= s`.
5. If `S` is empty:
	1. Let `b` be the greatest value in `V[x]`.
	2. The command is given an `INVALID VERSION` reply. The reply indicates version `x.b`. **Processing stops here.**
6. Let `m` be the the minimum element of `S`.
7. Implementation `x.m` services the command, and this version number is used on replies to the command. **Processing stops here.**

*Example:* For example, an "RSVPico" service implements versions `1.0`, `1.2`, and `2.0`. Thus,

	V = {1 -> {0, 2}, 2 -> {0}}

With the aforementioned example service:

* Citizens can send commands with version `1.0`. These commands will be fulfilled by the service's implementation of `1.0`. Replies will indicate version `1.0`. 
* Citizens can also send commands with either version `1.1` or `1.2`. These commands will be fulfilled by the service's implementation of `1.2`. Replies will indicate version `1.2`.
* Citizens can also send commands with version `2.0`.  These commands will be fulfilled by the service's implementation of `2.0`. Replies will use version `2.0`.
* Citizens can also send commands with no version specified.  These commands will be fulfilled by the service's implementation of `2.0`. Replies will use version `2.0`.
* Citizens can not successfully send commands with version `1.5`. These commands will not be processed. Replies will use the mnemonic `INVALID VERSION` and version `1.2`.
* Citizens can not successfully send commands with version `0.1` nor `2.1` nor `3.0`. These commands will not be processed. Replies will use the mnemonic `INVALID VERSION` and version `2.0`.