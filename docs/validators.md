# Validator

Hedwig requires a validator to validate message contents. Libraries come with some providers in-built, and also allow
plugging in your own custom validator.

## JSON Schema

The schema file must be a JSON-Schema [draft v4](http://json-schema.org/specification-links.html#draft-4) schema. 
There's a few more restrictions in addition to being a valid schema:

- There must be a top-level key called `schemas`. The value must be an object.
- `schemas`: The keys of this object must be message types.  The value must be an object.
- `schemas/<message_type>`: The keys of this object must be major version patterns for this message type. The
  value must be an object.
- `schemas/<message_type>/<major_version>.*`: This object must represent the data schema for given message type, and
  major version. Any minor version updates must be applied in-place, and must be non-breaking per semantic
  versioning.
- `schemas/<message_type>/<major_version>.*/x-version`: `<major>.<minor>` semver that represents the latest version 
  for this schema.

Note that the schema file only contains definitions for major versions. This is by design since minor version MUST be
backwards compatible.

Here's an example schema:
```json
{
    "id": "https://hedwig.standard.ai/schema",
    "$schema": "http://json-schema.org/draft-04/schema#",
    "description": "Example Schema",
    "schemas": {
        "user-created": {
            "1.*": {
                "description": "A new user was created",
                "type": "object",
                "x-version": "1.0",
                "required": [
                    "user_id"
                ],
                "properties": {
                    "user_id": {
                        "$ref": "https://hedwig.standard.ai/schema#/definitions/UserId/1.0"
                    }
                }
            }
        },
        "user-updated": {
            "1.*": {
                "description": "A user was updated",
                "type": "object",
                "x-version": "1.1",
                "required": [
                    "user_id"
                ],
                "properties": {
                    "user_id": {
                        "$ref": "https://hedwig.standard.ai/schema#/definitions/UserId/1.0"
                    }
                }
            }
        }
    },
    "definitions": {
        "UserId": {
            "1.0": {
                "type": "string"
            }
        }
    }
}
```

## Protobuf

The proto file must be a proto2 schema and must be pre-compiled by the application. There's a few more restrictions in
addition to being a valid schema:

- `<message_type>V<major_version>`: For every message type and every major version for that message type, a protobuf
  message with this name must be defined.
- Every protobuf message must include option `(hedwig.message_options).major_version` (defined [here](https://github.
  com/cloudchacho/hedwig/blob/main/protobuf/options.proto)).
- Every protobuf message may include option `(hedwig.message_options).minor_version` (defined [here](https://github.
  com/cloudchacho/hedwig/blob/main/protobuf/options.proto)), which defaults to 0.

Note that the schema file only contains definitions for major versions. This is by design since minor version MUST be
backwards compatible.

Here's an example schema:
```proto
syntax = "proto2";

package hedwig_examples;

import "cloudchacho/hedwig/protobuf/options.proto";

option go_package = "github.com/cloudchacho/hedwig-go/examples;main";

message UserCreatedV1 {
  option (hedwig.message_options).major_version = 1;
  option (hedwig.message_options).minor_version = 0;

  optional string user_id = 1;
}

message UserUpdatedV1 {
  option (hedwig.message_options).major_version = 1;
  option (hedwig.message_options).minor_version = 1;

  optional string user_id = 1;
}
```
