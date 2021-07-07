# Hedwig Library Spec

> Status: WIP

This document describes the spec for how Hedwig libraries ought to be implemented.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 (https://tools.ietf.org/html/rfc2119).

While this document’s status is WIP, upon any conflict between this spec and the Python library (https://github.
com/cloudchacho/hedwig-python), the latter will supersede this spec.

There are method names mentioned in this document - the semantics for such methods is the requirement, not the method name.

Features

1. Consuming messages from configured transport backends
2. Publishing messages to configured transport backends
3. Validating incoming and outgoing Hedwig messages
4. Managing dead-letter queues

Requirements

1. *General guidelines* - library SHOULD:
    1. Make interface as simple as possible
        1. For example, move configuration to a separate single API call instead of passing it around in every function.
    2. Have unit test coverage of >90%
    3. Provide ready-to-copy example code, quick start documentation, and migration guide for any breaking versions.
    4. Use semver (https://semver.org/) for library versions
        1. Breaking changes are acceptable with a major version change.
        2. ALL public methods are part of the library API and changes to any interfaces SHOULD be considered a breaking change.
2. *Message Versioning*
    1. All Hedwig messages are versioned using major.minor versioning scheme.
    2. The library MUST enforce and respect this versioning scheme.
3. *Payload Format* - what’s sent on the wire
    1. The payload sent on the wire needs to include additional “meta attributes” that allow the library to decode the data. List of meta attributes:
        1. format_version - version of the container schema. Format: <major>.<minor>
        2. headers - custom headers, flattened
        3. id - message identifier
        4. message_timestamp - message creation timestamp as epoch milliseconds.
        5. publisher - name of the publishing service
        6. schema - message schema. Format:
            1. JSON schema: schema reference, e.g. https://hedwig.example.com/schema#/schemas/trip.created/1.0
            2. Protobuf: <message_type>/<version>, e.g. trip.created/1.0
            3. FlatBuffers: <message_type>/<version>, e.g. trip.created/1.0
    2. For transport backends that support “message attributes” (for example, SNS message attributes (https://docs.aws.amazon.com/sns/latest/dg/sns-message-attributes.html)), meta attributes MUST be published as message attributes on the transport backend with a prefix of hedwig_, i.e. format_version meta attribute is sent as hedwig_format_version message attribute. The payload on the wire MUST be exactly the data provided by the application. Custom headers must be sent as message attributes as well, but MUST be verified to not conflict with library’s namespace (hedwig_).
    3. For transport backends that don’t support message attributes, library MUST provide a “container payload” mechanism. When using container payload, the meta attributes (described above) are encoded in the payload on the wire in addition to the data provided by the application. Payload MUST follow the schema defined here:
        1. JSON schema (jsonschema/container.json)
        2. Protobuf (protobuf/container.proto)
        3. FlatBuffers (flatbutters/container.fbs)
4. *Message Model* - the primary model for representing an incoming or outgoing message.
    1. MUST provide these interfaces:
        1. new - create new outgoing message
        2. extend_visibility_timeout - extends timeout for a message if worker needs more time
    2. MAY provide these interfaces:
        1. publish - publish the message
        2. serialize - serialize the message with appropriate payload AND attributes for transport
        3. deserialize - deserialize a payload and attributes from transport into model
    3. MUST provide these properties:
        1. id - message identifier - provisioned by the publisher
        2. type - message type - string or enum
        3. version - schema version
        4. data - payload
        5. timestamp - message creation timestamp
        6. publisher - name of the publishing service
        7. headers - custom headers
5. *Consumer* - the primary entry point for consuming messages
    1. MUST validate payload before calling the callback
    2. MUST provide these interfaces:
        1. listen_for_messages - primary interface for starting consumer loop. This MAY be blocking.
            1. If blocking, must allow a mechanism to cleanly stop the consumer
            2. MUST receive messages from multiple queues if the transport backend doesn’t support fan-out.
            3. Messages MUST only be acknowledged / deleted once processed successfully.
6. *Publisher* - the primary entry point for publishing messages
    1. MUST provide these interfaces:
        1. publish - publishes a validated message
7. *Transport Backend*
    1. Transport backends SHOULD be customizable with an API interface that lets users plug their own backend
    2. MUST validate payload before publishing
    3. MUST allow passing a bag of attributes in addition to the message payload
    4. MAY provide these interfaces:
        1. fetch_and_process_messages - primary method that starts consumer loop
        2. pre_process_hook_kwargs - customize pre process hooks
        3. post_process_hook_kwargs - customize post process hooks
        4. process_message - processes an individual queue message
        5. ack_message / nack_message - acknowledge / delete a queue message
        6. requeue_dead_letter - requeue DLQ messages back into the main queue
        7. extend_visibility_timeout - extends visibility timeout for an message if worker needs more time
        8. _mock_queue_message - creates a mock message for use in sync mode
    5. In-built transport backends that library MAY provide (at least one is a MUST):
        1. Google Pub/Sub
        2. AWS SNS/SQS
        3. AWS Lambda - only for consumers
8. *Validator*
    1. Validator SHOULD be customizable with an API interface that lets users plug their own validator
    2. MUST validate schema upon initialization
    3. MUST enforce major / minor versioning schema for messages
        1. For example, library MUST enforce that schema name includes major version
    4. MAY provide these interfaces:
        1. serialize - serialize the message with appropriate payload AND attributes for transport
        2. deserialize - deserialize a payload and attributes from transport into model
    5. In-build validators that library MAY provide (at least one is a MUST):
        1. JSON-Schema
        2. Protobuf
        3. FlatBuffers
9. *Settings and Configuration -* library MUST provide a way to configure these settings in some way:
    1. publisher / consumer backend class (as well as any additional configuration that backend classes may require)
    2. validator class (as well as any additional configuration that validator classes may require)
    3. credentials and project / account info for the backend
    4. HTTP/gRPC timeouts
    5. callbacks
        1. This MUST be a mapping from message type and major version or major version pattern to a callable
        2. e.g. {('user-created', '1.*'): 'callbacks.user_created_handler'}
    6. routing
        1. This MUST be a mapping from message type and major version or major version pattern to a topic name
        2. e.g. {('user-created', '1.*'): 'dev-user-created-v1'}
    7. hooks (if this feature is supported)
    8. sync mode (if this feature is supported)
    9. publisher (name of the publisher)
    10. use data container - (legacy libraries only) flag that allows turning off data containarization in favor of 
        transport message attributes.
10. *Sync mode* - library MAY support changing operation mode
    1. A flag that allows changing behavior of the library to bypass the actual transport in the cloud, and directly
       call consumer code. This is useful for testing consumer app behavior.
11. *Test Utilities -* library MAY provide these test utilities:
    1. Mock transport backend that simply collects all the messages in-memory for later verification.
    2. Null transport backend that simply ignores any publish requests.
    3. Factory classes / modules for easily creating Model objects.
    4. Fixtures / functions to verify code published messages as expected
12. *Hooks* - library MAY provide these hooks for easy customization:
    1. Pre-process - called right after receiving message from the transport and before doing any kind of processing on
       it
    2. Post-process - called right before sending message to the transport and after all processing has been done.
    3. Default headers - called when processing message for publishing to customize a set of headers for all messages.

Libraries

* Python: https://github.com/cloudchacho/hedwig-python
* Golang: https://github.com/cloudchacho/hedwig-go
* Rust: https://github.com/standard-ai/hedwig-rust
