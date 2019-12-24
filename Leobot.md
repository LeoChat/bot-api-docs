# LeOBot API

LeoBot is a chat-bot engine, specially built to automate insurance related conversations. With LeoBot you can do things like:

- Upsell / renew / update policies.
- Provide customer support.
- Run surveys.
- Collect analytics.
- Contact real agents.

Each instance of LeOBot can be customized on demand by the LeOBot team and will be given on-going support. In addition, LeOBot exposes a RESTful API for more abstract needs. An access token and a [Tenant ID](#tenant) need to be obtained in order to access the API and they should be provided with each and every request, e.g.:

    curl -H 'Accept: application/json' -H "Authorization: Bearer {token}" https://bot.meetleo.co/{tenant}/{resource}

Keep in mind that a single Tenant can have multiple tokens, of which permissions can vary (contact the LeOBot team for further information).

## Docs

The following routes are exposed by the API, grouped by their context:

- [Agent](#agent)

    - [**GET** &nbsp; /agents](#get-agents)
    - [**GET** &nbsp; /agent/{id}](#get-agentid)
    - [**PUT** &nbsp; /agent/{id}](#put-agentid)

- [Conversation](#conversation)

    - [**GET** &nbsp; /conversations/{agentId}](#get-conversationsagentid)
    - [**GET** &nbsp; /conversation/{id}](#get-conversationid)

- [Message](#message)

    - [**GET** &nbsp; /messages/{conversationId}](#get-messagesconversationid)
    - [**GET** &nbsp; /message/{id}](#get-messageid)

- [Event](#event)

    - [**GET** &nbsp; /events/{conversationId}](#get-eventsconversationid)
    - [**GET** &nbsp; /event/{id}](#get-eventid)

LeoBot also exposes [Webhooks](#webhooks) to keep track of the following publications:

- [conversationStarted(agentId)](#conversationstartedagentid)
- [conversationEnded(agentId)](#conversationendedagentid)
- [messageSent(conversationId)](#messagesentconversationid)
- [eventEmitted(conversationId)](#eventemittedconversationid)

To get a better understanding of the system and its relationships, here's a rough diagram that describes it, followed by an in-depth explanation about each and every entity:

![leo-schema](https://user-images.githubusercontent.com/7648874/71329161-3d912d00-252a-11ea-83b0-02f5d5ec1e71.png)

### Tenant

- **`id` (UUID)** - Tenant ID.
- **`agentsIds` (UUID[])** - The IDs of the [Agents](#agent) associated with this Tenant.

A single organization should be represented by a single Tenant. A Tenant can be seen as the root segment that connects all relevant data to its belonging organization, such as Conversations, Messages, or even access tokens. Each and every Tenant is completely distinct, and has absolutely nothing to do with one another.

### Agent

- **`id` (UUID)** - Agent ID.
- **`avatar` (string)** - An image URI for the Agent's avatar.
- **`name` (string)** - The name of the Agent. This will appear as the title in the chat room.
- **`description` (string)** - A short description of the Agent. This will appear as the subtitle in the chat room.
- **`theme` (string)** - A HEX value, RGB, or a color literal that will determine the color scheme of the chat room. Will affect elements such as the navigation bar, send-button, etc.
- **`conversationsIds` (UUID[])** - The IDs of the [Conversations](#conversation) associated with this Agent.

An Agent represents an instance of a bot that is programmed to handle a specific Conversation flow under a single Tenant. For example, one Agent can handle car policy and the other can handle pension policy. The Agent's properties will also affect the visuals of the chat room, such as the title, subtitle, and color scheme.

#### GET /agents

- **`agents` (Result, Partial [Agent](#agent)[])** - Tenant's Agents, without `conversationsIds`.

Gets all Agents records listed under the current Tenant, without their correlated Conversations. (Available only with "master" access token.

#### GET /agents/{id}

- **`id` (Param, UUID)** - Target Agent ID.
- **`agent` (Result, [Agent](#agent))** - Target Agent record.

Gets a full Agent record given its ID. (Available only with "master" access token.

### Conversation

- **`id` (UUID)** - Conversation ID.
- **`messagesIds` (UUID[])** - The IDs of the Conversation's [Messages](#message).
- **`eventsIds` (UUID[])** - The IDs of the [Events](#event) emitted throughout the Conversation.
- **`startedAt` (DateTime)** - The time at which the user has entered the Conversation for the first time.
- **`endedAt` (DateTime)** - The time at which the Conversation flow has ended.

A Conversation represents an exchange of Messages between the human and the bot. Each Conversation has a very specific set of Event names that can be emitted from it, and it needs to be known in advance. Contact the LeOBot team to get the Events which are relevant to your organization. For more information, see [reference for Event](#event).

#### GET /conversations

- **`agentId` (Param, UUID)** - Parent Agent ID.
- **`startedAfter` (Query, DateTime)** - Retrieve all Conversations started after the specified time.
- **`startedBefore` (Query, DateTime)** - Retrieve all Conversations started before the specified time.
- **`endedAfter` (Query, DateTime)** - Retrieve all Conversations ended after the specified time.
- **`endedBefore` (Query, DateTime)** - Retrieve all Conversations ended before the specified time.
- **`conversations` (Result, Partial [Conversation](#conversation)[])** - Agent's Conversations, without `eventsIds` or `messagesIds`.

Gets all Conversations records listed under the agent of the token, without their correlated Events or Messages.

#### GET /conversations/{id}

- **`id` (Param, UUID)** - Target Conversation ID.
- **`conversation` (Result, [Conversation](#conversation))** - Target Conversation record.

Gets a full Conversation record given its ID.

### Message

- **`id` (UUID)** - Message ID.
- **`sentAt` (DateTime)** - The time at which the Message was sent.
- **`senderRole` (MessageSender)** - Represents the one who sent the message. Can either be `BOT` or `HUMAN`.
- **`type` (MessageType)** - The type of the message. Can be one of:
    - *`TEXT`* - Represents a text Message. If this value is used, the `contents` field will be of type `string`.
    - *`ATTACHMENT`* - Represents a Message bound to an external file URL. If this value is used, the `contents` field will be of type `string`.
    - *`JSON`* - Represents a JSON message. If this value is used, the `contents` field will be of type `JSON`.
- **`contents` (string | JSON)** - The contents of the Message whose type will change according to the `type` field.

A Message represents a payload that was sent by a human or a bot throughout a Conversation. Note that a Message doesn't necessarily has to represent text, but can also represent an attachment, or a JSON (see `type` field).

#### GET /conversations/{conversationId}/messages

- **`conversationId` (Param, UUID)** - Parent Conversation ID.
- **`sentAfter` (Query, DateTime)** - Retrieve only Messages that were sent after the specified time.
- **`sentBefore` (Query, DateTime)** - Retrieve only Messages that were sent before the specified time.
- **`senderRole` (Query, MessageSender)** - Retrieve only Messages which were sent by the specified sender.
- **`types` (Query, MessageType[])** - Retrieve only Messages of the given types.
- **`messages` (Result, [Message](#message)[])** - Conversation's Messages.

Gets all Messages records listed under the given Conversation.

#### GET /conversations/{conversationId}/messages/{id}

- **`id` (Param, UUID)** - Target Message ID.
- **`conversationId` (Param, UUID)** - Target Conversation ID.

Gets a full Message record given its ID.

### Event

- **`id` (UUID)** - Event ID.
- **`name` (string)** - Event name.
- **`emittedAt` (DateTime)** - The time at which the Event was emitted throughout the conversation.
- **`payload` (JSON, optional)** - The payload bound to the Event when it was emitted.

An Event object represents a piece of an information that was emitted throughout the Conversation at a specific time so we can have more information about the Conversation's progress, e.g. when did a specific dialog start, what information did the user provide us with, etc. Events are mostly useful when used with [Webhooks](#webhook).

#### GET /conversations/{conversationId}/events

- **`conversationId` (Param, UUID)** - Parent Conversation ID.
- **`emittedAfter` (Query, DateTime)** - Retrieve only Messages that were emitted after the specified time.
- **`emittedBefore` (Query, DateTime)** - Retrieve only Messages that were emitted before the specified time.
- **`events` (Result, [Message](#message)[])** - Conversation's Events.

Gets all Events records listed under the given Conversation.

#### GET /conversations/{conversationId}/events/{id}

- **`id` (Param, UUID)** - Target Event ID.
- **`conversationId` (Param, UUID)** - Target Conversation ID.

Gets a full Event record given its ID.

## Webhooks

You can subscribe to Webhooks with the following endpoints.

- [**POST** &nbsp; /webhook/{id}]() - Registers a new subscription.
- [**DELETE** &nbsp; /webhook/{id}]() - Deletes a subscription.
- [**PUT** &nbsp; /webhook/{id}]() - Updates a subscription.

A payload must be provided and will always contain:

- **`callbackUri` (string)** - The callback URI of the subscription.
- **`payload` (JSON, optional)** - A payload object that defines the subscription.

With Webhooks you can subscribe to the following publications:

#### conversationStarted(agentId)

- **`agentId` (Payload, UUID)** - Parent Agent ID.
- **`conversation` (Result, Partial Conversation)** - Target Conversation, without `eventsIds` or `messagesIds`.

Published each time a new Conversation with the given Agent was started.

#### conversationEnded(agentId)

- **`agentId` (Payload, UUID)** - Parent Agent ID.
- **`conversation` (Result, Partial [Conversation](#conversation))** - Published Conversation, without `eventsIds` or `messagesIds`.

Published each time a Conversation with the given Agent was ended.

#### messageSent(conversationId)

- **`conversationId` (Payload, UUID)** - Parent Conversation ID.
- **`message` (Result, [Message](#message))** - Published Message.

Published each time a Message was sent in the given Conversation.

#### eventEmitted(conversationId)

- **`conversationId` (Payload, UUID)** - Parent Conversation ID.
- **`event` (Result, [Event](#event))** - Published Event.

Published each time an Event was emitted throughout the given Conversation.

&nbsp;
*© 2019 LeO All Rights Reserved*
