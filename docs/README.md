# Hedwig

Hedwig is an inter-service communication bus that works on cloud providers such as AWS and GCP, while keeping things
pretty simple and straight forward.

It allows validation of the message payloads before they are sent, helping to catch cross-component incompatibilities
early.

Hedwig allows separation of concerns between consumers and publishers so your services are loosely coupled, and the
contract is enforced by the message payload validation. Hedwig may also be used to build asynchronous APIs.

Read more about messaging systems concepts [here](concepts).

## Getting started

To get started with Hedwig, choose your [backend](/hedwig/backends) and provision the infra resources. Then, choose a
[validator](/hedwig/validators) and define your schema as per validator requirements. Start a consumer 
application and a publisher application. Finally, publish a message from the publisher, and you should see it arrive in 
the consumer.

## Implementations

- [Python](https://github.com/cloudchacho/hedwig-python)
- [Golang](https://github.com/cloudchacho/hedwig-go)
- [Rust](https://github.com/standard-ai/hedwig-rust)
