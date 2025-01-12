# CML - Common Messaging Layer 

CML is an inversion of typical APIs for services, intended to be driven by pull mechanics, or rather, the API coming to you. All messaging is done over protobufs and relies heavily on the generated serialization and deserialiation of these messages.

# Usage

## Producer
A **Producer** produces requests that will be fulfilled by **Providers**.  **Producers** do this by creating a *Demand* for information, that specifies both the Messge type they're offering and the message type they'll accept as a response.  Both of which are derrived from protobufs.

```ts

// Example of loading the certificate using CML
// This only needs to be done once per application
const certificate = readFileSync('./my-service.pem', 'utf-8');
const privateKey = readFileSync('./my-service.key', 'utf-8');

CML.Config.setCredentials({
  certificate,
  privateKey,
});

// When running in a development mode you can disable cert checks like this:
CML.Config.setNoCAMode()

// Optionally you can change the multicast address used for Service Discovery
CML.Config.setMulticast(224.0.0.1);

// Producer Side (Demand Queue):
const demand = new Demand<Messages.MyMessage, Messages.MyResponse>({
  desiredProviders: 3,
  maxProviders: 5
});

demand.handler(async (workItem, results) => {
    // handle completed request, or a failed request.
    // results will now be of type MyResponse, if results are returned.
    console.log("received results for work item", workItem, results);
});
demand.announce();

// create a work item, to be processed by others
const request: Messages.MyMessage = {foo: "bar", retries: 3};
demand.addWorkItem(request);
```
## Provider

A **Provider** provides *Capabilities* to handle various *Demand*s, and will automatically balance itself when providing these capabilities. It may help to think of these in terms of being endpoints, as they return unitary data to satisfy *Demand*s.
```ts

// Example of loading the certificate using CML
// This only needs to be done once per application
const certificate = readFileSync('./my-service.pem', 'utf-8');
const privateKey = readFileSync('./my-service.key', 'utf-8');

CML.Config.setCredentials({
  certificate,
  privateKey,
});

// When running in a development mode you can disable cert checks like this:
CML.Config.setNoCAMode()

// Optionally you can change the multicast address used for Service Discovery
CML.Config.setMulticast(224.0.0.1);

// Provider Side (Capabilities):
// This will connect to a service that provides this type, and return the result
const myCapability = new Capability<Messages.MyMessage, Messages.MyResponse>(async (workItem) => {
    // process the data, and then send a result back
    const results: Messages.MyResponse = {status: "OK", message: "success"}; 
    return results;
});

```
