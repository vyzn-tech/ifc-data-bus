# IFC data bus v0.2

## The Challenge

**IFC data bus 0.2** is a challenge of the **AEC/O Hackathon 2025 – Munich Edition**.

<a href="ifc-data-bus challenge Munich.pdf">The Challenge Presentation (.pdf)</a>

## Problem Statement

Architects and planners rely on a broad ecosystem of software tools, yet **poor data integration between these tools creates major bottlenecks** in the design process. This fragmentation prevents holistic project optimization and slows down coordination. Although OpenBIM aims to support interoperability, it still lacks robust mechanisms for **bi-directional data exchange**—particularly when it comes to **re-integrating data into CAD and CDE environments**. Existing initiatives often do not prioritize full OpenBIM compliance, leaving a clear opportunity for alternative, standards-aligned solutions.

At the same time, the emerging **modularized and layerized IFC5** standard introduces new capabilities for more flexible, fine-grained, and machine-friendly data handling. These advances open the door to new workflows built around decentralized, event-driven data exchange rather than monolithic file transfers.

## Goal

Develop a proof-of-concept for an **IFC Data Bus** that demonstrates decentralized data synchronization using **IFC5**, **MQTT**, and **IDS**, leveraging the **modularity and layered structure of IFC5** to enable more scalable and targeted data flows.  

The system should consist of **four interoperating components** that **publish, process, and read model data** across the data bus.  

The entire solution should be delivered as an easy-to-run **docker-compose** setup and published on GitHub for straightforward adoption.

Last but not least, IFC data bus needs a logo :)

## Resources

All provided resources are from the **IFC 4x world** and require thoughtful **adoption to IFC5**.

1. <a href="https://drive.google.com/file/d/1G6e2UCSyiQRj5Dxday6ytkK4VL-FCIHI/view?usp=sharing">Test IFC File</a>
2. <a href="https://github.com/vyzn-tech/ifc-data-bus/tree/main/poc/message">Message format</a>
3. Message Flow:

```mermaid
sequenceDiagram
    participant A as System A<br/>(AuthoringTool)
    participant B as System B<br/>(CoordinationPlatform)
    participant C as System C<br/>(LCAService)
    participant D as System D<br/>(Viewer/UI)
    participant Bus as Message Bus / Topic<br/>(IFC5-Message Layer)

    A->>Bus: Model.Publish<br/>correlationId=init-publish-001<br/>revision=1
    Bus-->>B: Model.Publish
    Bus-->>C: Model.Publish
    Bus-->>D: Model.Publish

    A->>Bus: Model.Change<br/>correlationId=change-001<br/>revision=2
    Bus-->>B: Model.Change
    Bus-->>C: Model.Change
    Bus-->>D: Model.Change

    B->>Bus: Analysis.LCA.Request<br/>correlationId=LCA-request-001<br/>standard=DIN EN 15978:2012-10
    Bus-->>C: Analysis.LCA.Request

    C->>C: Perform LCA Calculation
    C-->>Bus: Analysis.LCA.Result<br/>correlationId=LCA-request-001
    Bus-->>D: Analysis.LCA.Result

    D->>D: Combine model + results<br/>Visualize output

```