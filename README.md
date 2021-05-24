# Hedwig

Hedwig is a inter-service communication bus that works on AWS and GCP, while keeping things pretty simple and straight forward.

It allows validation of the message payloads before they are sent, helping to catch cross-component incompatibilities early.

Hedwig allows separation of concerns between consumers and publishers so your services are loosely coupled, and the contract is enforced by the message payload validation. Hedwig may also be used to build asynchronous APIs.
