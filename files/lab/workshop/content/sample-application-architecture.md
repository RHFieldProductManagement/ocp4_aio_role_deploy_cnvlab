So far we've deployed some basic workloads and shown you the main interfaces for OpenShift Virtualization, but now we're going to move into a more realistic "real-world" scenario. This lab section introduces you to the architecture of the **ParksMap** application used throughout this workshop, to get a better understanding of the things you'll be doing from a *developer* perspective. ParksMap is a polyglot geo-spatial data visualisation application built using the microservices architecture and is composed of a set of services which are developed using different programming languages and frameworks, roughly resembling the following architecture:

<img  border="1" align="center" src="img/roadshow-app-architecture.png" title="Application architecture"/>

The main service is a web application which has a server-side component in charge of aggregating the geo-spatial APIs provided by multiple independent backend services and a client-side component in JavaScript that is responsible for visualising the geo-spatial data on the map. The client-side component which runs in your browser communicates with the server-side via WebSockets protocol in order to update the map in real-time.

There will be a set of independent backend services deployed that will provide different mapping and geo-spatial information. The set of available backend services that provide geo-spatial information are:

* WorldWide National Parks
* Major League Baseball Stadiums in North America

The original source code for this application is available [here](https://github.com/openshift-roadshow/parksmap-web). The server-side component of the ParksMap web application acts as a communication gateway to all the available backends. These backends will be dynamically discovered by using service discovery mechanisms provided by OpenShift which will be discussed in more details in the following labs. The backend applications use *MongoDB* to persist map and geo-spatial information. In order to showcase how containers and virtual machines can run together in an OpenShift Environment, you will be deploying MongoDB applications as virtual machines.

Select "**Deploying Parksmap**" to continue.