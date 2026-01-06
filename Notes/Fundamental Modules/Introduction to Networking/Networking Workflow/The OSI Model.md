# The OSI Model

The purpose of defining the **ISO/OSI standard** was to create a **clear reference model** that allows different technical systems to communicate with each other, regardless of the underlying devices or technologies in use. Instead of focusing on specific implementations, the OSI model gives you a structured way to **visualise and reason about how network communication works**.

The model is built around **seven layers**, arranged hierarchically. Each layer represents a specific phase in establishing and maintaining a connection. When data is sent across a network, it passes through these layers in order, which makes it easier for you to trace *how* a connection is formed and *where* problems may occur.

---

## OSI Layers and Their Roles

| Layer | Name         | What This Layer Does                                                                                                    |
| ----- | ------------ | ----------------------------------------------------------------------------------------------------------------------- |
| 7     | Application  | Controls data input and output and provides network services directly to applications.                                  |
| 6     | Presentation | Translates data into a format that is independent of the application and system (for example, encoding and encryption). |
| 5     | Session      | Manages the logical connection between systems and handles session setup, maintenance, and teardown.                    |
| 4     | Transport    | Provides end-to-end data control, including segmentation, flow control, and congestion handling.                        |
| 3     | Network      | Handles logical addressing and routes packets across networks from sender to receiver.                                  |
| 2     | Data Link    | Ensures reliable, error-free transmission over the physical medium by framing data.                                     |
| 1     | Physical     | Transmits raw bits using electrical, optical, or electromagnetic signals over wired or wireless media.                  |

---

## How the Layers Are Grouped

To make the model easier to reason about, the layers are often grouped by purpose:

* **Layers 2–4**: Transport-oriented layers
  These focus on moving data reliably and efficiently across the network.

* **Layers 5–7**: Application-oriented layers
  These focus on managing sessions and presenting data to applications in a usable form.

Each layer has **clearly defined responsibilities** and well-defined interfaces to the layers above and below it. A layer provides services to the layer above while relying on services from the layer below. This separation is what makes the model so powerful.

---

## How Communication Actually Happens

When two systems communicate, **all seven layers are involved on both sides**:

* The **sending system** processes data from **Layer 7 down to Layer 1**
* The **receiving system** processes the data from **Layer 1 up to Layer 7**

This means the OSI model is effectively traversed **twice** for every communication: once during transmission and once during reception.

At each layer, specific tasks are performed to ensure:

* Security
* Reliability
* Correct delivery
* Acceptable performance

---

## Encapsulation and Decapsulation

When an application sends data:

1. The data starts at **Layer 7 (Application)**
2. Each lower layer adds its own control information
3. By the time it reaches **Layer 1 (Physical)**, the data is ready to be transmitted as raw bits

On the receiving side, this process is reversed:

* Each layer removes and processes its own information
* The data is passed upward until the application can finally use it

This step-by-step wrapping and unwrapping is what allows complex communication to remain manageable.

---

## Why This Matters to You

You do not need to memorise the OSI model for the sake of theory. You use it as a **thinking tool**.

Whenever something breaks, ask yourself:

* Is this a physical issue?
* Is routing failing?
* Is the service misbehaving?
* Is the data being presented incorrectly?

If you can map a problem to a specific layer, troubleshooting becomes faster and more precise. As you move deeper into networking, traffic analysis, and exploitation, the OSI model becomes less abstract and more like a **mental checklist** you use without even noticing.
