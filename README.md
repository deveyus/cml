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
# General Design

1. **WebTransport Connection Management:**
    * **Connection Establishment:**
        * Handles creating and managing WebTransport sessions with other services or clients based on URLs.
        * Handles secure connection setup, including any necessary authentication or authorization.
        * Implements necessary mechanisms for connection timeouts, retries, and handling connection failures.
        * Robust error handling and reconnection logic is implemented to allow nodes to come and go. 

    * **Stream Management:**
        * Handles creating, managing, and closing WebTransport streams for data transfer (as all data is sent on a stream).
        * Manages bidirectional streams.

    * **Session Management:**
        * Implements logic to ensure that connections are managed appropriately, and re-established if they are lost, or if a new connection is available.

    * **Message Handling:**
        * Handles message segmentation, reassembly, and provides an interface for sending messages on streams.
        * Implements logic for dealing with timeouts.
        * Implements logic for message prioritization or guarantees, if those are needed.

2. **Message Serialization/Deserialization:**
    * **Protocol Buffer Integration:**
        * Integrates with generated Protocol Buffer code for serializing message objects to byte streams before sending over WebTransport, and decoding byte streams into strongly typed objects when receiving messages.
        * Implements logic to retrieve the type of message from the type itself, so that routing can take place.

    * **Validation:**
        * Implements a mechanism to validate that the received data matches its schema.
        * Provides a mechanism for sending actionable and helpful error messages.

3. **Decentralized Service Discovery:**
    * **Multicast Handling:**
        * Listens for multicast messages on the defined address.
        * Has a mechanism to generate appropriate messages to send over multicast.

    * **Service Discovery Logic:**
        * Parses the content of the discovery broadcasts to extract information about the service.
        * Maintains an internal list of known services and their corresponding WebTransport endpoints.
        * Implements logic for adding, removing, and updating the lists of neighbors.

    * **WebTransport Connection:**
        * Connects to services using WebTransport, as determined by the discovery information.
        * Implements mechanisms for re-establishing connections when they are closed, or fail.

    * **Peer to Peer Updates:**
        * Manages sending messages to known neighbors, to update service lists.

4.  **Pull-Based Work Management:**
    *   **`Demand` (Producer) Implementation:**
        *   Creates a WebTransport endpoint for accepting provider connections, and exposes that endpoint through the service discovery mechanism.
        *  Implements message routing, for when results are received.
        *   Manages pending work items with timeouts.
        *  Implements a mechanism to set the number of desired, and max, connections, and how that is distributed to other services.
    *   **`Capability` (Provider) Implementation:**
        *   Connects to available `Demand` endpoints.
        *  Prioritizes connection to `Demand` endpoints based on lowest network latency, and also based on those queues with the largest deficit on their Desired Providers. After that it will use those that have the highest deficit on Max Providers.
        *   Pulls work items from the "queues."
        *   Sends results back to the producer with the tickets.

5. **Load Balancing (Implicit in pull mechanism):**
    * **Pull-Based Nature:** The pull-based system inherently allows for a form of load balancing as services will only process data when they are ready to do so.

    * **Desired Providers:** Will be used to help prioritize which services should connect to each other first.

    * **Max Providers:** Ensures that no Demand is overburdened, and that a finite number of services can connect to any given Demand.


     *  When a provider connects to a demand, it should store the network latency of that connection.

    * **Prioritization:** The system will then prioritize lower latency connections, but also those connections that are more in demand, calculating a score based on the available providers for a specific service, and using the function `score = sqrt(providerDeficit * latency)`.


6. **Error Handling and Logging:**
    * **WebTransport Errors:** Handle connection failures, session failures, stream failures, and other errors.
    * **Protocol Buffer Errors:** Handle errors during serialization or deserialization.
    * **Application Errors:** Handle errors during message processing and route those errors back to the original source.
    * **Centralized Logging:** Ensure that there is a single logging system that is shared between all of the services to ensure that error messages and other information can be easily captured.

7. **Configuration and Management:**
  * Configuration is handled largely at runtime, and done through the CML.Conffig.  This is to be explicit and not pollute enviroment variables.
