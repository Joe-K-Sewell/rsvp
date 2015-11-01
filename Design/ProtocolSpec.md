# Restaurant Suggestion and Voting Protocol

**VERSION 0.1**

This specification is a public API that adheres to [Semantic Versioning](http://semver.org/).

## Introduction

The Restaurant Suggestion and Voting Protocol (RSVP) allows members of an organization (an *electorate*) to nominate and vote on dining locations for a lunch trip. Deliberately over-engineered and has a user-facing syntax of HTTP. Solve the pesky human problem of your coworkers being indecisive or apathetic about sustenance by injecting *technology*!

## Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](http://tools.ietf.org/html/rfc2119).

### Entities

* **Server** - An automated service that organizes elections and communicates with citizens (over RSVP) to produce election results.
	* An **implementation** - Unless otherwise qualified, refers to an implementation of a Server. 
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

## Protocol vs. Implementation

This protocol does not specify how citizens communicate with the server, only the content of that communication. While conforming implementations SHOULD allow the textual representations specified here, they may also include other means of communication, such as a graphical interface.

## Commands

Commands are issued in the form:

	(sentinel) (verb) [(path value)]
	[(option_name): (option value)]
	[(option_name): (option value)]
	[...]

* **Sentinel** - A key phrase introducing the command.
* **Verb** - The specific kind of command to issue.
* **Path** - An argument for the command, such as a location name.
* **Options** - Additional arguments for some commands.

### Sentinel

A sentinel is used to indicate the text being issued is relevant to RSVP. This is necessary in shared interactive environments, such as chat rooms, where not all text posted to the room needs to be taken as input by the server. All implementations SHOULD support sentinels, but depending on the use case they might not require them.

Sentinels take the form:

	RSVP[/(major).(minor)]

where `major` and `minor` correspond to the major and minor version number of the protocol version the command expects.

Version information is OPTIONAL, but if it is specified, the server is required to validate that it supports that version. For instance, the following are all valid sentinels for a server supporting both `1.0` and `1.1`:

`RSVP`  
`RSVP/1.0`  
`RSVP/1.1`  

If the version given is not supported by the server, the server MUST reply `505 RSVP Version Not Supported` and MUST NOT carry out the command.

### Options

### Verbs

In addition to the verbs specified here, implementations may define other verbs to be used, including as aliases of protocol-specified verbs.

#### SUGGEST

Issued by a citizen to indicate a location should be considered in the current election.

* **Path**: Required. The name of the location.
* **Options**: None protocol-defined. May include information about why the location is being proposed, or stipulations about its inclusion.

The exact result of a citizen suggesting a location is dependent on the ES, but typically it will add the suggested location to a list of candidates that can then be voted on. Because suggestions may be issued in private channels, the ES should decree when a new suggestion is added, or decree the full list at a regular interval.

*Examples*

Suggesting a location without options:

	RSVP/0.1 SUGGEST Joe's

Suggesting a location with options:

	RSVP/0.1 SUGGEST Big Dipper
	Why: Group special, one day only
	When: Before noon, they're usually crowded

#### VOTE

#### RESCIND

#### REGISTER (citizen)

#### REGISTER (location)

#### METADATA

#### OPTIONS

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
* **Body** - Human-readable information contained in the decree.

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

## Plugin Interface

## Future Development