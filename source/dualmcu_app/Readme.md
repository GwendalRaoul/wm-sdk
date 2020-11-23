# Introduction

The Wirepas Mesh stack (hereafter referred to as the "stack") provides services
for the application layer (hereafter referred to as the "application"). The
services are exposed via Service Access Points ("SAPs"). The SAPs are divided
into Data SAP (DSAP), Management SAP (MSAP), and Configuration SAP (CSAP). The
SAP services are provided in the form of primitives and SAP data is exposed as
attributes.  
All field lengths are in octets (i.e. units of eight bits), unless otherwise
stated.

## Service Access Points

The SAPs provide the following general services:

-   **DSAP**: Provides methods for data transfer to and from the stack (and the
    network)

-   **MSAP**: Provides methods for transferring stack management information and
    reading/writing management attributes. Management attributes provide
    information of the run-time state of the stack and are valid only when the
    stack is running.

-   **CSAP:** Provides methods for reading/writing stack configuration
    attributes. Configuration attributes can only be written when the stack is
    stopped.

Currently, the SAPs are realized as a Universal Asynchronous
Receiver/Transmitter (UART) serial interface.

## Primitive Types

The primitives are divided into four classes: request, confirm, indication, and
response. The general usage of the primitives is as follows (Also see Figure 1):

-   A **request** is issued by the application when it wants to use a stack
    service.

-   A **confirm** is a reply from the stack to the request issued by the
    application.

-   An **indication** is issued by the stack when it has data/information it
    wants to send to the application. In the point of view of the application,
    indications are asynchronous messages from the stack.

-   A **response** is a reply from the application to the indication issued by
    the stack.

![](media/8972836bfe5347ac7602af603c814d5d.png)

  
*Figure 1. Primitive usage in the communication between the application and the
stack*

  
Three different use cases can be identified for the above primitives:

1.  Application issues commands to the stack or needs to send data/information.

    1.  Application issues the appropriate **request** primitive.

    2.  The stack responds with corresponding **confirm** primitive to
        acknowledge the request.

2.  Application queries data/information from the stack and the stack responds
    immediately:

    1.  Application issues the appropriate **request** primitive.

    2.  The stack responds with corresponding **confirm** primitive containing
        the requested data/information.

3.  The stack needs to send asynchronous data/information to the application:

    1.  The stack generates appropriate **indication(s)**.

    2.  The stack asserts the Interrupt ReQuest ("IRQ") signal to notify the
        application that it has one or more pending indications.

    3.  The application queries the indications from the stack and acknowledges
        every indication with corresponding **response** primitive.

**Note 1:** Some application requests may generate an immediate response with
which the stack informs that the request has been taken for processing and in
addition optional indication with which the stack informs that the request has
actually been processed.

  
**Note 2:** The stack indications are always notified via IRQ and can be queried
by the application. The stack never sends data/information to the application on
its own without the application explicitly requesting it. This enables the
application to have full control over the communication between the application
and the stack, and offers full flexibility on the application architecture
(interrupt-based/polling) and scheduling (application can sleep and run its own
tasks when it wants to and communicate with the stack when it wants to). In an
extreme case, e.g. when application MCU pin count is too low, the IRQ signal can
even be omitted and the indication queries can be sent periodically, though this
implementation is not the most energy-efficient nor provides lowest delay
depending on the query interval.

## Attributes

Attributes are small pieces of data that affect the way the stack works, or are
used to inform the application of the state of the stack. Before the stack can
be started in normal operation, a few critical attributes need to be configured
properly (see section [“Required
Configuration”](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#Required-Configuration)).  
Attributes can either be read-only, readable and writable, or write-only. The
attributes can also be persistent or non-persistent. If the attribute is
persistent, its value will be retained over device power downs and stack stops,
i.e. the value of an attribute is stored in non-volatile memory. Otherwise, the
attribute value will be lost when the device is powered down or the stack
stopped.  
Note: Although there are no strict restrictions on how often a persistent
variable can be updated by the application layer, each update causes a tiny bit
of wear on the non-volatile memory. If a persistent variable is to be updated
periodically, updating it less often than once every 30 minutes is recommended.

## Serial Interface Specification

The physical interface between the application MCU and stack MCU is Universal
Asynchronous Receiver/Transmitter ("UART"), colloquially called a serial port.
The data is exchanged in frames.

## General Frame Format

The serial frames have similar frame separation mechanism as in SLIP (RFC 1055).
SLIP framing works as follows:

-   Two octet values are reserved: 0xC0, called "END" and 0xDB, called "ESC".

-   A frame begins and ends with octet 0xC0 (END).

-   Any octet of value 0xC0 (END) within the frame is encoded as 0xDB (ESC),
    0xDC.

-   Any octet of value 0xDB (ESC) within the frame is encoded as 0xDB (ESC),
    0xDD.

-   Any other octet is passed through as-is.

In addition, two additional END octets are used to wake up the stack side UART
when starting communication. These END octets are present only when
communicating towards the stack UART. The stack UART will transmit a single END
octet in the beginning of a frame.  
The general format of the serial frame is presented in Figure 2. Note that the
different primitives and corresponding content of the payload (thick border in
Figure 2) are specified in section “[Stack Service
Specification](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#Stack-Service-Specification)”.
The meaning of the different frame fields is described in Table 1.

![](media/6cd3ffd5e5a13fde0f737972d4d6df74.png)

  
*Figure 2. General format of the serial frame*

  
Table 1. General serial frame fields

| **Field**        | **Size** | **Description**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
|------------------|----------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| *END*            | 1        | Frame separator, octet 0xC0.  Starts and ends a SLIP encoded frame. In addition two extra END-octets are used to wake up the stack side UART when starting communication.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| *Primitive ID*   | 1        | The identified of the used primitive. Different primitives and their primitive identifiers are specified in section [Stack Service Specification](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#Stack-Service-Specification).  As a general rule:  Initiating primitives (request-primitives from the application side and indication-primitives from the stack side) have always the most significant bit set to 0.  Responding primitives (confirm-primitives from the stack side and response-primitives from the application side) always have the most significant bit set to 1.  Confirm.primitive_id = 0x80 request.primitive_id  Response.primitive_id = 0x80 indication.primitive_id |
| *Frame ID*       | 1        | Frame identifier. The initiating peer decides the ID and responding peer uses the same value in the response frame:  The application decides the Frame ID for a request-primitive and the stack sends corresponding confirm-primitive with the same Frame ID.  The stack decides the Frame ID for an indication-primitive and the application sends corresponding response-primitive with the same Frame ID.                                                                                                                                                                                                                                                                                                                                            |
| *Payload length* | 1        | The following payload length in octets, excluding the CRC octets.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| *Payload*        | *N1*     | The payload of the frame, depends on the primitive in question.  Different primitives and corresponding content of the payload are specified in section [Stack Service Specification](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#Stack-Service-Specification).                                                                                                                                                                                                                                                                                                                                                                                                                             |
| *CRC*            | 2        | Checksum over the whole frame that has not been SLIP encoded, excluding the CRC octets. When receiving a frame, the SLIP encoding is removed and the CRC is calculated over the decoded frame. When sending a frame, the CRC is calculated first and SLIP encoding is employed after that.                                                                                                                                                                                                                                                                                                                                                                                                                                                              |

**Note**: These fields are used only locally for the communication between the
application MCU and the stack. They are not actually transmitted on the network.

## Flow Control

The application MCU is the master in the communication between the application
and the stack.  
The stack UART receiver is enabled by two wake up symbols (as described in
section [General Frame
Format](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#General-Frame-Format))
and any octets received via UART are processed in an Interrupt Service Routine
(ISR). Thus, no serial interface flow control is required when communicating to
the stack (flow control may be needed in upper level if the stack memory runs
low due to congestion).  
Communication is always initiated by the application MCU. Thus, no serial
interface flow control is required in the application direction either.  
The stack informs pending indications via an IRQ signal. The IRQ signal is
active low. When the stack has pending indications, the IRQ is asserted, i.e.
the IRQ signal is pulled down. When the stack does not have pending indications,
the IRQ is not asserted, i.e. the IRQ signal is held high.  
The usage of request-confirm and indication-response pairs should always be
atomic. This means, that a new request should not be sent before a confirmation
is received for a previous request (application initiated communication) and a
new indication should not be sent before a response is received for a previous
indication (stack initiated communication).

## UART Configuration

UART configuration is defined and configurable in the Dual MCU application.
Default Dual MCU application settings are described in in Table 2.

*Table 2. UART default configuration*

| **Parameter**       | **Value**     |
|---------------------|---------------|
| Baud rate           | 125000 bps    |
| Number of data bits | 8             |
| Parity              | No parity bit |
| Number of stop bits | 1             |

## Endianness and Bit Order

Multi-octet fields are transferred least significant octet first (i.e.
little-endian).  
Octets are transferred most significant bit first.

## Timing

There is a reception timeout for received UART frames. A transmission of
complete API frame to the stack MCU shall take no longer than 100 ms.

## CRC Calculation (CRC-16-CCITT)

The used CRC type is CRC-16-CCITT. See Annex A for example implementation and
test vectors.

# Stack Service Specification

The different services provided by the stack are specified in this section. The
specification includes description of usage, primitives, and frame formats of
the services. Table 3 list all the primitives and their primitive IDs.

*Table 3. All primitives and their primitive IDs*

| **SAP** | **Primitive**                      | **Primitive ID** |
|---------|------------------------------------|------------------|
| DSAP    | DSAP-DATA_TX.request               | 0x01             |
|         | DSAP-DATA_TX.confirm               | 0x81             |
|         | DSAP-DATA_TX_TT.request            | 0x1F             |
|         | DSAP-DATA_TX_TT.confirm            | 0x9F             |
|         | DSAP-DATA_TX.indication            | 0x02             |
|         | DSAP-DATA_TX.response              | 0x82             |
|         | DSAP-DATA_RX.indication            | 0x03             |
|         | DSAP-DATA_RX.response              | 0x83             |
| MSAP    | MSAP-INDICATION_POLL.request       | 0x04             |
|         | MSAP-INDICATION_POLL.confirm       | 0x84             |
|         | MSAP-STACK_START.request           | 0x05             |
|         | MSAP-STACK_START.confirm           | 0x85             |
|         | MSAP-STACK_STOP.request            | 0x06             |
|         | MSAP-STACK_STOP.confirm            | 0x86             |
|         | MSAP-STACK_STATE.indication        | 0x07             |
|         | MSAP-STACK_STATE.response          | 0x87             |
|         | MSAP-APP_CONFIG_DATA_WRITE.request | 0x3A             |
|         | MSAP-APP_CONFIG_DATA_WRITE.confirm | 0xBA             |
|         | MSAP-APP_CONFIG_DATA_READ.request  | 0x3B             |
|         | MSAP-APP_CONFIG_DATA_READ.confirm  | 0xBB             |
|         | MSAP-APP_CONFIG_DATA_RX.indication | 0x3F             |
|         | MSAP-APP_CONFIG_DATA_RX.response   | 0xBF             |
|         | MSAP-NRLS.request                  | 0x40             |
|         | MSAP-NRLS.confirm                  | 0xC0             |
|         | MSAP-NRLS_STOP.request             | 0x41             |
|         | MSAP-NRLS_STOP.confirm             | 0xC1             |
|         | MSAP-NRLS_STATE_GET.request        | 0x42             |
|         | MSAP-NRLS_STATE_GET.response       | 0xC2             |
|         | MSAP-NRLS_GOTOSLEEP_INFO.request   | 0x4C             |
|         | MSAP-NRLS_GOTOSLEEP_INFO.response  | 0xCC             |
|         | MSAP-ATTRIBUTE_WRITE.request       | 0x0B             |
|         | MSAP-ATTRIBUTE_WRITE.confirm       | 0x8B             |
|         | MSAP-ATTRIBUTE_READ.request        | 0x0C             |
|         | MSAP-ATTRIBUTE_READ.confirm        | 0x8C             |
|         | MSAP-GET_NBORS.request             | 0x20             |
|         | MSAP-GET_NBORS.confirm             | 0xA0             |
|         | MSAP-SCAN_NBORS.request            | 0x21             |
|         | MSAP-SCAN_NBORS.confirm            | 0xA1             |
|         | MSAP-SCAN_NBORS.indication         | 0x22             |
|         | MSAP-SCAN_NBORS.response           | 0xA2             |
|         | MSAP-SINK_COST_WRITE.request       | 0x38             |
|         | MSAP-SINK_COST_WRITE.confirm       | 0xB8             |
|         | MSAP-SINK_COST_READ.request        | 0x39             |
|         | MSAP-SINK_COST_READ.confirm        | 0xB9             |
|         | MSAP-SCRATCHPAD_START.request      | 0x17             |
|         | MSAP-SCRATCHPAD_START.confirm      | 0x97             |
|         | MSAP-SCRATCHPAD_BLOCK.request      | 0x18             |
|         | MSAP-SCRATCHPAD_BLOCK.confirm      | 0x98             |
|         | MSAP-SCRATCHPAD_STATUS.request     | 0x19             |
|         | MSAP-SCRATCHPAD_STATUS.confirm     | 0x99             |
|         | MSAP-SCRATCHPAD_UPDATE.request     | 0x1A             |
|         | MSAP-SCRATCHPAD_UPDATE.confirm     | 0x9A             |
|         | MSAP-SCRATCHPAD_CLEAR.request      | 0x1B             |
|         | MSAP-SCRATCHPAD_CLEAR.confirm      | 0x9B             |
|         | MSAP-MAX_QUEUE_TIME_WRITE.request  | 0x4F             |
|         | MSAP-MAX_QUEUE_TIME_WRITE.confirm  | 0xCF             |
|         | MSAP-MAX_QUEUE_TIME_READ.request   | 0x50             |
|         | MSAP-MAX_QUEUE_TIME_READ.confirm   | 0xD0             |
| CSAP    | CSAP-ATTRIBUTE_WRITE.request       | 0x0D             |
|         | CSAP-ATTRIBUTE_WRITE.confirm       | 0x8D             |
|         | CSAP-ATTRIBUTE_READ.request        | 0x0E             |
|         | CSAP-ATTRIBUTE_READ.confirm        | 0x8E             |
|         | CSAP-FACTORY_RESET.request         | 0x16             |
|         | CSAP-FACTORY_RESET.confirm         | 0x96             |

**Note:** The general framing follows the format described in section [General
Frame
Format](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#General-Frame-Format).
For clarity, the figures of the frames presented in this section also include
the general frame fields (Primitive ID, Frame ID, Payload length, and CRC), but
their descriptions are omitted as they are already explained in section [General
Frame
Format](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#General-Frame-Format).

## Node Addressing

The Wirepas Mesh Dual-MCU API services use a 32-bit address to indicate sources
and destinations of packets.  
Two special addresses have been reserved. First, address 0x0000 0000 (zero) or
0xFFFFFFFE (4 294 967 294) is used as the *anySink* address which identifies
that the source or the destination of a packet is an unspecified sink on the
network. The highest address 0xFFFF FFFF is used as the *broadcast* address. It
is used to transmit a downlink packet to all nodes on the network from a sink.  
Nodes are not allowed to use these two special addresses as their own address.
Similarly, addresses in multicast address space cannot be used as own address.  
The addresses are summarized in Table 4.

  
*Table 4. Addressing summary*

| **Address type** | **Valid address space**                                                                                     | **Description**                                                                                                                                                                                                                                                                                                                             |
|------------------|-------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Unicast          | 0x00000001-  0x7FFFFFFF  (1 – 2 147 483 647)  and  0x81000000-  0xFFFFFFFD  (2 164 260 864 – 4 294 967 293) | Valid unicast addresses. Each node on the network must have one of these addresses set as its address. Two or more devices with identical addresses should never be present on a network.                                                                                                                                                   |
| Broadcast        | 0xFFFF FFFF  (4 294 967 295)                                                                                | Broadcast address with which a packet is delivered to all nodes on the network                                                                                                                                                                                                                                                              |
| AnySink          | 0xFFFFFFFE  (4 294 967 294)  or 0x00000000 (0)                                                              | Address which identifies that the source or the destination of a packet is an unspecified sink on the network  With current tree routing, only nodes may use this as the destination address when sending packets. These addresses are reserved as Wirepas reserved addresses and cannot be used as addresses for any nodes in the network. |
| Multicast        | 0x80000000-  0x80FFFFFF  (2 147 483 648 –  2 164 260 863)                                                   | Packet is delivered to the group of nodes. Group may contain 0 or more nodes. Each node may belong to 0 or more groups. The lowest 24 bits contain the actual group address and highest bit is an indication that message is sent to that group.                                                                                            |

## Data Services (DSAP)

The data services are used to transmit/receive application data via the network.

### DSAP-DATA_TX Service

The DSAP-DATA_TX service is used to transport APDUs from the application to the
stack. The stack transmits the APDUs to other node(s) on the network, according
to set parameters. The DSAP-DATA_TX service includes the following primitives:

-   DSAP-DATA_TX.request

-   DSAP-DATA_TX.confirm

-   DSAP-DATA_TX_TT.request

-   DSAP-DATA_TX_TT.confirm

-   DSAP-DATA_TX.indication

-   DSAP-DATA_TX.response (All response primitives have the same format, see
    section [Response
    Primitives](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#Response-Primitives))

#### DSAP-DATA_TX.request

The DSAP-DATA_TX.request is issued by the application when it wants to send
data. The DSAP-DATA.request frame is depicted in Figure 3.

![](media/0048bbb0d46932a56b6fd76fe95c6443.png)

  
*Figure 3. DSAP-DATA_TX.request frame*

  
The DSAP-DATA_TX.request frame fields (solid border in the figure) are explained
in Table 5

*Table 5. DSAP-DATA_TX.request frame fields*

| **Field Name**        | **Size** | **Valid Values**                             | **Description**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
|-----------------------|----------|----------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| *PDUID*               | 2        | 0 – 65534                                    | PDU identifier decided by the application  The PDU ID can be used to keep track of APDUs processed by the stack as the same PDU ID is contained in a corresponding DSAP-DATA_TX.indication sent by the stack to the application. E.g. the application can keep the PDU in its own buffers until the successful transmission is indicated by the stack in DSAP-DATA_TX.indication.  PDU ID 65535 (0xFFFF) is reserved and should not be used.  Also see Note 1.                                                                                                                                                                                                                                                                                |
| *SourceEndpoint*      | 1        | 0 – 239                                      | Source endpoint number  Also see Note 2.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| *DestinationAddress*  | 4        | 0 – 4294967295                               | Destination node address  Also see Note 3.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| *DestinationEndpoint* | 1        | 0 – 239                                      | Destination endpoint number  Also see Note 2.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| *QoS*                 | 1        | 0 or 1                                       | Quality of service class to be used. The different values are defined as follows:  0 = Use traffic class 0, i.e. normal priority 1 = Use traffic class 1, i.e. high priority.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| *TXOptions*           | 1        | 00xx xxxx  (bitfield, where x can be 0 or 1) | The TX options are indicated as a bit field with individual bits defined as follows:   Bit 0 = 0: Do not generate DSAP-DATA_TX.indication  Bit 0 = 1: Generate DSAP-DATA_TX.indication  Bit 0 is used to register for receiving a DSAP-DATA_TX.indication after the PDU has been successfully transmitted to next hop node or cleared from the PDU buffers due to timeout or congestion. Also see Note 1.   Bit 1 = 1, Use unacknowledged CSMA-CA transmission method  Bit 1 = 0, Use normal transmission method.  See Note 4.   Bits 2-5: Hop limit. Maximum number of hops executed for packet to reach the destination. See Note 5.   Bits 6-7: Reserved  Here, bit 0 is the least significant bit and bit 7 is the most significant bit.  |
| *APDULength*          | 1        | 1 – 102                                      | The length of the following APDU in octets                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| *APDU*                | 1 – 102  | \-                                           | Application payload                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |

**Note 1:** These fields are used only locally for the communication between the
application layer and the stack. They are not actually transmitted on the
network. Only 16 requests where generation of DSAP-DATA_TX.indication is active
are allowed at the same time. Without generation of the indication, there is
room for plenty of more requests simultaneously.

  
**Note 2:** The endpoint numbers are used to distinguish different application
channels. E.g. if the device has multiple sensors it could use different
endpoint numbers for APDUs containing data from different sensors or different
endpoint numbers for applications with different functionality.  
Endpoints 240 – 255 are reserved for Wirepas Mesh stack internal use.  
Note 3:Note that a broadcast will only be transmitted (downlink) to the nodes
directly under the sink's routing tree. To reach all nodes on the network, it is
necessary to send the broadcast from all sinks. All devices can send traffic to
themselves (loopback) by using their own address as destination.

  
**Note 4:** The unacknowledged CSMA-CA transmission method can be used in a
mixed network (i.e. network consisting of both CSMA-CA and TDMA devices) by
CSMA-CA device originated packets transmission only to CSMA-CA devices. The
purpose of this method is to avoid a performance bottleneck by NOT transmitting
to TDMA devices. Also, if used with sink-originated transmissions (by CSMA-CA
mode sinks), the throughput is better when compared to a 'normal' transmission,
however there is some penalty in reliability (due to unacknowledged nature of
transmission).

  
**Note 5:** Hop limit sets the upper value to the number of hops executed for
packet to reach the destination. By using hop limiting, it is possible to limit
the distance how far the packet is transmitted to and avoiding causing
unnecessary traffic to network. Hop count value of 0 is used to disable the hop
limiting. Hop limiting value does **not** have any impact when using *AnySink*
address as destination node address but is discarded.

#### DSAP-DATA_TX.confirm

The DSAP-DATA_TX.confirm is issued by the stack as a response to the
DSAP-DATA_TX.request. The DSAP-DATA_TX.confirm frame is depicted in Figure 4.

![](media/6988b6b69e23e035da4e673a7f204cf7.png)

  
*Figure 4. DSAP-DATA_TX.confirm frame*

  
The DSAP-DATA_TX.confirm frame fields (solid border in the figure) are explained
in Table 6.

  
*Table 6. DSAP-DATA_TX.confirm frame fields*

| **Field Name** | **Size** | **Valid Values** | **Description**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
|----------------|----------|------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| *PDUID*        | 2        | 0 – 65534        | PDU identifier set by the application in the corresponding DSAP-DATA_TX.request  This field is only used for data TX requests where an indication is requested, i.e. TX options bit 0 is set (see field TXOptions in section [DSAP-DATA_TX.request](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#DSAP-DATA_TX.request)). If no indication is requested, the value of this field is undefined.                                                                                                                                                                                                                                   |
| *Result*       | 1        | 0 – 10           | The return result of the corresponding DSAP-DATA_TX.request. The different values are defined as follows:  0 = Success: PDU accepted for transmission 1 = Failure: Stack is stopped 2 = Failure: Invalid QoS-parameter 3 = Failure: Invalid TX options-parameter 4 = Failure: Out of memory  5 = Failure: Unknown destination address 6 = Failure: Invalid APDU length-parameter 7 = Failure: Cannot send indication 8 = Failure: PDUID is already in use  9 = Failure: Invalid src/dest end-point  10 = Failure: Access denied (see section [cFeatureLockBits](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#cFeatureLockBits)) |
| *Capacity*     | 1        | \-               | Number of PDUs that still can fit in the PDU buffer (see section [mPDUBufferCapacity](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#mPDUBufferCapacity) for details)                                                                                                                                                                                                                                                                                                                                                                                                                                                             |

#### DSAP-DATA_TX_TT.request

The DSAP-DATA_TX_TT.request is identical to the DSAP-DATA_TX.request, except
there is one extra field for setting buffering delay to an initial no-zero
value. The DSAP-DATA.request frame is depicted in Figure 5.

  


![](media/d4fa1fd5470893261af903e5e13b6572.png)

  
*Figure 5. DSAP-DATA_TX_TT.request frame*

  
The DSAP-DATA_TX_TT.request frame fields (solid border in the figure) are
explained in Table 7.

*Table 7. DSAP-DATA_TX_TT.request frame fields*

| **Field Name**        | **Size** | **Valid Values**                             | **Description**                                                                                                                                                                 |
|-----------------------|----------|----------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| *PDUID*               | 2        | 0 – 65534                                    | See description in chapter [DSAP-DATA_TX.request](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#DSAP-DATA_TX.request) |
| *SourceEndpoint*      | 1        | 0 – 239                                      | See description in chapter [DSAP-DATA_TX.request](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#DSAP-DATA_TX.request) |
| *DestinationAddress*  | 4        | 0 – 4294967295                               | See description in chapter [DSAP-DATA_TX.request](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#DSAP-DATA_TX.request) |
| *DestinationEndpoint* | 1        | 0 – 239                                      | See description in chapter [DSAP-DATA_TX.request](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#DSAP-DATA_TX.request) |
| *QoS*                 | 1        | 0 or 1                                       | See description in chapter [DSAP-DATA_TX.request](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#DSAP-DATA_TX.request) |
| *TXOptions*           | 1        | 00xx xxxx  (bitfield, where x can be 0 or 1) | See description in chapter [DSAP-DATA_TX.request](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#DSAP-DATA_TX.request) |
| *BufferingDelay*      | 4        | 0 – 4 294 967 295                            | The time the PDU has been in the application buffers before it was transmitted over API.  Expressed in units of 1/128th of a second.                                            |
| *APDULength*          | 1        | 1 – 102                                      | See description in chapter [DSAP-DATA_TX.request](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#DSAP-DATA_TX.request) |
| *APDU*                | 1 – 102  | \-                                           | See description in chapter [DSAP-DATA_TX.request](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#DSAP-DATA_TX.request) |

#### DSAP-DATA_TX_TT.confirm

The DSAP-DATA_TX_TT.confirm is issued by the stack as a response to the
DSAP-DATA_TX_TT.request. It is identical to DSAP-DATA_TX.confirm, explained in
section
[DSAP-DATA_TX.confirm](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#DSAP-DATA_TX.confirm).

#### DSAP-DATA_TX.indication

The DSAP-DATA_TX.indication is issued by the stack as an asynchronous reply for
the DSAP-DATA_TX.request after the APDU of the corresponding
DSAP-DATA_TX.request is successfully transmitted to next hop node or cleared
from stack buffers due to timeout or congestion. The DSAP-DATA_TX.indication is
sent only if the application registers it in the corresponding
DSAP-DATA_TX.request's TX options parameter. The DSAP-DATA_TX.indication frame
is depicted in Figure 6.

![](media/71dbe5d9e8b14459465fbf72c4a04ce8.png)

  
*Figure 6. DSAP-DATA_TX.indication frame*

  
The DSAP-DATA_TX.indication frame fields (solid border in the figure) are
explained in Table 8.

*Table 8. DSAP-DATA_TX.indication frame fields*

| **Field Name**        | **Size** | **Valid Values** | **Description**                                                                                                                                                                  |
|-----------------------|----------|------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| *IndicationStatus*    | 1        | 0 or 1           | 0 = No other indications queued1 = More indications queued                                                                                                                       |
| *PDUID*               | 2        | 0 – 65534        | PDU identifier set by the application in the corresponding DSAP-DATA_TX.request                                                                                                  |
| *SourceEndpoint*      | 1        | 0 – 239          | Source endpoint number                                                                                                                                                           |
| *DestinationAddress*  | 4        | 0 – 4294967295   | Destination node address set by the application in the corresponding DSAP-DATA_TX.request                                                                                        |
| *DestinationEndpoint* | 1        | 0 – 239          | Destination endpoint number                                                                                                                                                      |
| *BufferingDelay*      | 4        | \-               | The time the PDU has been in the stack buffers before it was transmitted. Reported in units of *BufferingDelay / 128* seconds i.e. *BufferingDelay \* 7.8125* milliseconds.      |
| *Result*              | 1        | 0 or 1           | The return result of the corresponding DSAP-DATA_TX.request. The different values are defined as follows:  0 = Success: PDU was successfully sent 1 = Failure: PDU was discarded |

### DSAP-DATA_RX Service

The DSAP-DATA_RX service supports the transport of received APDUs from the stack
to the application layer. The DSAP-DATA_RX service includes the following
primitives:

-   DSAP-DATA_RX.indication

-   DSAP-DATA_RX.response (All response primitives have the same format, see
    section [Response
    Primitives](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#Response-Primitives))

#### DSAP-DATA_RX.indication

The DSAP-DATA_RX.indication is issued by the stack when it receives data from
the network destined to this node. The DSAP-DATA_RX.indication frame is depicted
in Figure 7.

![](media/b0d1461af560770d392d0fa108a5f921.png)

  
*Figure 7. DSAP-DATA_RX.indication frame*

  
The DSAP-DATA_RX.indication frame fields (thick border in the figure) are
explained in Table 9.

  
*Table 9. DSAP-DATA_RX.indication frame fields*

| **Field Name**        | **Size** | **Valid Values** | **Description**                                                                                                                                                                                                                                                                                                                                                                                                                              |
|-----------------------|----------|------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| *IndicationStatus*    | 1        | 0 or 1           | 0 = No other indications queued1 = More indications queued                                                                                                                                                                                                                                                                                                                                                                                   |
| *SourceAddress*       | 4        | 0 – 4294967295   | Source node address                                                                                                                                                                                                                                                                                                                                                                                                                          |
| *SourceEndpoint*      | 1        | 0 – 239          | Source endpoint number                                                                                                                                                                                                                                                                                                                                                                                                                       |
| *DestinationAddress*  | 4        | 0 – 4294967295   | Destination node address                                                                                                                                                                                                                                                                                                                                                                                                                     |
| *DestinationEndpoint* | 1        | 0 – 239          | Destination endpoint number                                                                                                                                                                                                                                                                                                                                                                                                                  |
| *QoS + Hop count*     | 1        | 0 – 255          | Bits 0-1 (LSB): Quality of service class to be used. The different values are defined as follows:  0 = Use traffic class 0, i.e. normal priority 1 = Use traffic class 1, i.e. high priority   Bits 2-7: Hop count: how many hops were used to transmit the data to the destination (1-n hops)   For example, value 0x29 (0b00101001) tells that high priority data was received and ten hops were used to transmit data to the destination. |
| *TravelTime*          | 4        | \-               | Travel time of the PDU on the network. Reported in units of *TravelTime / 128* seconds i.e. *TravelTime \* 7.8125* milliseconds.                                                                                                                                                                                                                                                                                                             |
| *APDULength*          | 1        | \-               | The length of the following APDU in octets                                                                                                                                                                                                                                                                                                                                                                                                   |
| *APDU*                | 1 – 102  | \-               | Application payload                                                                                                                                                                                                                                                                                                                                                                                                                          |

## Management Services (MSAP)

The management services are used to control the stack at run-time, as well as
read and write run-time parameters of the stack.

### INDICATION_POLL Service

For enabling variety of application architectures, the application may poll the
stack indications when it is most convenient. This is enabled by the indication
IRQ with MSAP-INDICATION_POLL service. The MSAP-INDICATION_POLL service includes
the following primitives:

-   MSAP-INDICATION_POLL.request

-   MSAP-INDICATION_POLL.confirm

The MSAP-INDICATION_POLL service is used to query indications from the stack.
This mechanism is used for all indications independent of the service.  
The basic flow for receiving indications from the stack goes as follows:

1.  The stack asserts the IRQ signal to indicate that it has pending
    indication(s) that it wants to deliver to the application. For
    hardware-specific information on the IRQ signal, see the appropriate
    hardware reference manual.

2.  The application sends MSAP-INDICATION_POLL.request to query for the
    indication(s).

3.  The stack responds with MSAP-INDICATION_POLL.confirm to indicate that it
    will start sending pending indications.

4.  The stack sends a pending indication. The individual indication format
    depends on the service that has issued the indication and follows the
    indication formats specified in this document.

5.  The application sends a response to acknowledge the indication. In the
    response, the application also indicates if it wants to receive another
    pending indication.

6.  If the response frame indicated that the application wants to receive
    another indication and there are still pending indications: Go to step 4.

The indication exchange stops if a) there are no more pending indications (in
which case the stack de-asserts the IRQ), or b) the application indicates in a
response that it does not want to receive more indications at the moment (in
which case pending indications, if there are any, can be queried later).  
Note:If there are no pending indications when the application issues a
MSAP-INDICATION_POLL.request (i.e. the request is issued, but IRQ signal is not
asserted), the stack replies only with MSAP-INDICATION_POLL.confirm and informs
that there are no pending indications at the moment.

#### MSAP-INDICATION_POLL.request

The MSAP-INDICATION_POLL.request is issued by the application layer when it
wants to query stack indications. The MSAP-INDICATION_POLL.request frame is
depicted in Figure 8.

![](media/707c899f90c2480e9908439a2a98f320.png)

  
*Figure 8. MSAP-INDICATION_POLL.request frame*

  
The MSAP-INDICATION_POLL.request frame does not contain any payload.

#### MSAP-INDICATION_POLL.confirm

The MSAP-INDICATION_POLL.confirm is issued by the stack as a response to the
MSAP-INDICATION_POLL.request. The MSAP-INDICATION_POLL.confirm frame is depicted
in Figure 9.

![](media/fb4b6cfe805688bf36359404448c9cba.png)

  
*Figure 9. MSAP-INDICATION_POLL.confirm frame*

  
The MSAP-INDICATION_POLL.confirm frame fields (solid border in the figure) are
explained in Table 10.

  
*Table 10. MSAP-INDICATION_POLL.confirm frame fields*

| **Field Name** | **Size** | **Valid Values** | **Description**                                                                                                                                                                                                            |
|----------------|----------|------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| *Result*       | 1        | 0 or 1           | The return result of the corresponding MSAP-INDICATION_POLL.request. The different values are defined as follows:  1 = Pending indications exist and stack will start sending the indication(s) 0 = No pending indications |

### MSAP-STACK_START Service

The stack can be started using the MSAP-STACK_START service. The
MSAP-STACK_START service includes the following primitives:

-   MSAP-STACK_START.request

-   MSAP-STACK_START.confirm

#### MSAP-STACK_START.request

The MSAP-STACK_START.request issued by the application layer when the stack
needs to be started. The MSAP-STACK_START.request frame is depicted in Figure
10.

![](media/95904fdc313f2f9a75c7ac3e807db81b.png)

*Figure 10. MSAP-STACK_START.request frame*

  
The MSAP-STACK_START.request frame fields (solid border in the figure) are
explained in Table 11.

  
*Table 11. MSAP-STACK_START.request frame fields*

| **Field Name** | **Size** | **Valid Values**                   | **Description**                                                                                                                                                                                                                                                                                                                                                                                                    |
|----------------|----------|------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| *StartOptions* | 1        | 0000 000x  (where x can be 0 or 1) | The stack start options are indicated as a bit field with individual bits defined as follows:  Bit 0 = 0: Start stack with auto-start disabled, see Note, below Bit 0 = 1: Start stack with auto-start enabled Bit 1: Reserved Bit 2: Reserved Bit 3: Reserved Bit 4: Reserved Bit 5: Reserved Bit 6: Reserved  Bit 7: Reserved  , where bit 0 is the least significant bit and bit 7 is the most significant bit. |

**The MSAP-STACK_START.confirm issued by the stack as a response to the
MSAP-STACK_START.request. The MSAP-STACK_START.confirm frame is depicted in
Figure 11**.

#### MSAP-STACK_START.confirm

The MSAP-STACK_START.confirm issued by the stack as a response to the
MSAP-STACK_START.request. The MSAP-STACK_START.confirm frame is depicted in
Figure 11.

![](media/e72689426cd3fcf7c6ddcd9738083650.png)

*Figure 11. MSAP-STACK_START.confirm frame*

  
The MSAP-STACK_START.confirm frame fields are explained in Table 12.

  
*Table 12. MSAP-STACK_START.confirm frame fields*

| **Field Name** | **Size** | **Valid Values**                   | **Description**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
|----------------|----------|------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| *Result*       | 1        | 000x xxxx  (where x can be 0 or 1) | The return result of the corresponding MSAP-STACK_START.request. The result is indicated as a bit field with individual bits defined as follows:  0x00: Success: Stack started Bit 0 = 1: Failure: Stack remains stopped Bit 1 = 1: Failure: Network address missing Bit 2 = 1: Failure: Node address missing Bit 3 = 1: Failure: Network channel missing Bit 4 = 1: Failure: Role missing Bit 5 = 1: Failure: Application configuration data missing (valid only on sink device) Bit 6: Reserved  Bit 7 = 1: Failure: Access denied (see section [cFeatureLockBits](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#cFeatureLockBits))  , where bit 0 is the least significant bit and bit 7 is the most significant bit. |

### MSAP-STACK_STOP Service

The stack can be stopped using the MSAP-STACK_STOP service. The MSAP-STACK_STOP
service includes the following primitives:

-   MSAP-STACK_STOP.request

-   MSAP-STACK_STOP.confirm

#### MSAP-STACK_STOP.request

The MSAP-STACK_STOP.request is issued by the application layer when the stack
needs to be stopped. Stopping the stack will cause the firmware to reboot. This
has the following side effects:

-   All buffered data will be lost, including data being routed from other nodes

-   If there is a new OTAP scratchpad that has been marked to be processed, the
    bootloader will process it, e.g. update the stack firmware (see section
    [MSAP-SCRATCHPAD
    Services](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#MSAP-SCRATCHPAD-Services))

Note:A successful MSAP-STACK_STOP.request sets the MSAP auto-start attribute to
disabled. For more information on the auto-start feature, see section
[mAutostart](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#mAutostart).  
The MSAP-STACK_STOP.request frame is depicted in Figure 12.

![](media/1b93ff13070d39f98f5b8d59c79ac2b9.png)

  
*Figure 12. MSAP-STACK_STOP.request frame*

  
The MSAP-STACK_STOP.request frame does not contain any payload.

#### MSAP-STACK_STOP.confirm

The MSAP-STACK_STOP.confirm issued by the stack as a response to the
MSAP-STACK_STOP.request. The MSAP-STACK_STOP.confirm frame is depicted in Figure
13.

![](media/b41a5ea09b76007e2557552735edae17.png)

  
*Figure 13. MSAP-STACK_STOP.confirm frame*

  
The MSAP-STACK_STOP.confirm frame fields are explained in Table 13.

  
*Table 13. MSAP-STACK_STOP.confirm frame fields*

| **Field Name** | **Size** | **Valid Values** | **Description**                                                                                                                                                                                                                                                                                                                                                      |
|----------------|----------|------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| *Result*       | 1        | 0, 1 or 128      | The return result of the corresponding MSAP-STACK_STOP.request. The different values are defined as follows:  0 = Success: Stack stopped 1 = Failure: Stack already stopped  128 = Failure: Access denied (see section [cFeatureLockBits](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#cFeatureLockBits)) |

**Note:** After device has sent the MSAP-STACK_STOP.confirm message, device is
rebooted. This takes a while. During the rebooting, the device does not respond
to messages. One example of how to detect when the rebooting has been done is to
issue read-only commands to the device and when it responses to such, the
rebooting has been performed. For example, use MSAP-ATTRIBUTE_READ command (see
section [MSAP-ATTRIBUTE_READ
Service](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#MSAP-ATTRIBUTE_READ-Service))
to query attribute
[mStackStatus](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#mStackStatus).

### MSAP-STACK_STATE Service

The MSAP-STACK_STATE service informs the stack state at boot. MSAP-STACK_STATE
service includes the following primitives:

-   MSAP-STACK_STATE.indication

-   MSAP-STACK_STATE.response (All response primitives have the same format, see
    section [Response
    Primitives](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#Response-Primitives))

#### MSAP-STACK_STATE.indication

The MSAP-STACK_STATE.indication is issued by the stack when it has booted. The
MSAP-STACK_STATE.indication frame is depicted in Figure 14.

![](media/e09977d0030313017c68bc62f7bf6547.png)

  
*Figure 14. MSAP-STACK_STATE.indication frame*

  
The MSAP-STACK_STATE.indication frame fields are explained in Table 14.

*Table 14. MSAP-STACK_STATE.indication frame fields*

| **Field Name**     | **Size** | **Valid Values**                   | **Description**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
|--------------------|----------|------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| *IndicationStatus* | 1        | 0 or 1                             | 0 = No other indications queued1 = More indications queued                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| *Status*           | 1        | 0xxx xxxx  (where x can be 0 or 1) | The stack status is indicated as a bit field with individual bits defined as follows:  Bit 0 = 0: Stack running, see Note below Bit 0 = 1: Stack stopped Bit 1 = 0: Network address set Bit 1 = 1: Network address missing Bit 2 = 0: Node address set Bit 2 = 1: Node address missing Bit 3 = 0: Network channel set Bit 3 = 1: Network channel missing Bit 4 = 0: Role set Bit 4 = 1: Role missing Bit 5 = 0: Application configuration data valid Bit 5 = 1: Application configuration data missing (valid only on sink device) Bit 7: Reserved  , where bit 0 is the least significant bit and bit 7 is the most significant bit. |

**Note:** If the stack sends an MSAP-STACK_STATE.indication where the status bit
0 = 0, it means that the stack has auto-started. If the status bit 0 = 1, it
means that auto-start is disabled. For more information on the auto-start
feature, see section
[mAutostart](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#mAutostart).

### MSAP-APP_CONFIG_DATA_WRITE Service

The MSAP-APP_CONFIG_DATA_WRITE service can be used for two things:

1.  Configure application-specific parameters (application configuration data)
    to the application running in the nodes (via the network)

2.  Configure transmission interval of the stack diagnostic data

The application configuration data is persistent global data for the whole
network. The data format can be decided by the application. Application
configuration data can be set by the sinks' application after which it is
disseminated to the whole network and stored at every node. It can include e.g.
application parameters, such as measurement interval. The service makes it
possible to set the data, after which every new node joining the network
receives the data from its neighbors without the need for end-to-end polling.
Furthermore, new configurations can be set and updated to the network on the
run.  
The MSAP-APP_CONFIG_DATA_WRITE service includes the following primitives:

-   MSAP-APP_CONFIG_DATA_WRITE.request

-   MSAP-APP_CONFIG_DATA_WRITE.confirm

**Note 1:**The MSAP-APP_CONFIG_DATA_WRITE service can only be used in sink role.  
**Note 2:** In a network including multiple sinks, the same configuration data
should be set to all sinks so that it can be guaranteed to disseminate to every
node.  
**Note 3:**Application configuration data is stored in permanent memory
similarly to the persistent attributes. To avoid memory wearing, do not write
new values too often (e.g. more often than once per 30 minutes).

#### MSAP-APP_CONFIG_DATA_WRITE.request

The MSAP-APP_CONFIG_DATA_WRITE.request is issued by the application when it
wants to set or change the network configuration data contents. The
MSAP-APP_CONFIG_DATA_WRITE.request frame is depicted in Figure 15.

![](media/d24832f98edc526739aeb690a9133f8d.png)

  
*Figure 15. MSAP-APP_CONFIG_DATA_WRITE.request frame*

  
The MSAP-APP_CONFIG_DATA_WRITE.request frame fields are explained in Table 15.

  
*Table 15. MSAP-APP_CONFIG_DATA_WRITE.request frame fields*

| **Field Name**           | **Size** | **Valid Values**                                   | **Description**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
|--------------------------|----------|----------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| *SequenceNumber*         | 1        | 0 – 254, default value is 0                        | Sequence number for filtering old and already received application configuration data packets at the nodes.  The sequence number must be increment by 1 every time new configuration is written, i.e. new diagnostic data interval and/or new application configuration data is updated. See section [Sequence Numbers](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#Sequence-Numbers) for details.  A sequence number that is the current value of existing application configuration data is invalid. A value of 255 is invalid. Therefore, after value of 254, the next valid value is 0. |
| *DiagnosticDataInterval* | 2        | 0, 30, 60, 120, 300, 600, 1800, default value is 0 | Diagnostic data transmission interval in seconds, i.e. how often the nodes on the network should send diagnostic PDUs.  If the value is 0, diagnostic data transmission is disabled.                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| *AppConfigData*          | X        | Raw hex data, default value is filled with 0x00    | Application configuration data. The format can be decided by the application.  Size of the field is defined by CSAP attribute cAppConfigDataSize (see section [cAppConfigDataSize](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#cAppConfigDataSize))                                                                                                                                                                                                                                                                                                                                         |

**Note:** It is recommended that the configuration data is not written too
often, as new configuration data is always written to the non-volatile memory of
the sink and disseminated to the network. This can cause unnecessary wearing of
the memory with devices that need to use the program memory to store persistent
variables and unnecessary load to the network.

#### MSAP-APP_CONFIG_DATA_WRITE.confirm

The MSAP-APP_CONFIG_DATA_WRITE.confirm is issued by the stack in response to the
MSAP-APP_CONFIG_DATA_WRITE.request. The MSAP-APP_CONFIG_DATA_WRITE.confirm frame
is depicted in Figure 16.

![](media/916b966a022e3a693942f0ac22ac71cf.png)

  
*Figure 16. MSAP-APP_CONFIG_DATA_WRITE.confirm frame*

  
The MSAP-APP_CONFIG_DATA_WRITE.confirm frame fields are explained in Table 17.

  
*Table 17. MSAP-APP_CONFIG_DATA_WRITE.confirm frame fields*

| **Field Name** | **Size** | **Valid Values** | **Description**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
|----------------|----------|------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| *Result*       | 1        | 0 – 4            | The return result of the corresponding MSAP-APP_CONFIG_DATA_WRITE.request. The different values are defined as follows:  0 = Success: New configuration written to sink's non-volatile memory and scheduled for transmission 1 = Failure: The node is not a sink  2 = Failure: Invalid DiagnosticDataInterval value 3 = Failure: Invalid SequenceNumber value  4 = Failure: Access denied (see section [cFeatureLockBits](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#cFeatureLockBits)) |

### MSAP-APP_CONFIG_DATA_READ Service

The MSAP-APP_CONFIG_DATA_READ service can be used to read the network-wide
application configuration data. The MSAP-APP_CONFIG_DATA_READ service includes
the following primitives:

-   MSAP-APP_CONFIG_DATA_READ.request

-   MSAP-APP_CONFIG_DATA_READ.confirm

For sinks, the service returns the configuration data last written to the stack
using the MSAP-APP_CONFIG_DATA_WRITE service. For nodes, the service returns the
configuration data that was last received from neighboring nodes.

#### MSAP-APP_CONFIG_DATA_READ.request

The MSAP-APP_CONFIG_DATA_READ.request is issued by the application when it wants
to read the network configuration data contents. The
MSAP-APP_CONFIG_DATA_READ.request frame is depicted in Figure 17.

![](media/a56b9d3ca8e41291d941bfbf36217d4a.png)

*Figure 17. MSAP-APP_CONFIG_DATA_READ.request frame*

  
The MSAP-APP_CONFIG_DATA_READ.request frame does not contain any payload.

#### MSAP-APP_CONFIG_DATA_READ.confirm

The MSAP-APP_CONFIG_DATA_READ.confirm is issued by the stack in response to the
MSAP-APP_CONFIG_DATA_READ.request. The MSAP-APP_CONFIG_DATA_READ.confirm frame
is depicted in Figure 18.

![](media/2b169d4da14a051491041fbcd6b4c807.png)

  
*Figure 18. MSAP-APP_CONFIG_DATA_READ.confirm frame*

  
The MSAP-APP_CONFIG_DATA_READ.confirm frame fields are explained in Table 18.

  
*Table 18. MSAP-APP_CONFIG_DATA_READ.confirm frame fields*

| **Field Name**           | **Size** | **Valid Values**               | **Description**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
|--------------------------|----------|--------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| *Result*                 | 1        | 0 – 2                          | Return result for the corresponding MSAP-APP_CONFIG_DATA_READ.request. The different values are defined as follows:  0 = Success: Configuration received/set 1 = Failure: No configuration received/set  2 = Failure: Access denied (see section [cFeatureLockBits](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#cFeatureLockBits))  When used with nodes, indicates whether configuration has been received from neighbors. When used with sinks, indicates whether configuration has already been set (by using MSAP-APP_CONFIG_DATA_WRITE service). |
| *SequenceNumber*         | 1        | 0 – 254                        | Sequence number for filtering old and already received application configuration data packets at the nodes. This parameter can be used by the application to decide if the configuration data has been updated. See section [Sequence Numbers](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#Sequence-Numbers) for details.  The returned value is never 255.                                                                                                                                                                                           |
| *DiagnosticDataInterval* | 2        | 0, 30, 60, 120, 300, 600, 1800 | Diagnostic data transmission interval in seconds, i.e. how often the stack should send diagnostic PDUs  If the value is 0, diagnostic data transmission is disabled.                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| *AppConfigData*          | X        | \-                             | Application configuration data. The format can be decided by the application.  Size of the field is defined by CSAP attribute cAppConfigDataSize (see section [cAppConfigDataSize](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#cAppConfigDataSize))                                                                                                                                                                                                                                                                                                   |

### MSAP-APP_CONFIG_DATA_RX Service

The MSAP-APP_CONFIG_DATA_RX service provides support for asynchronous sending of
received configuration data to the application when new data is received from
the network. The MSAP-APP_CONFIG_DATA_RX service includes the following
primitives:

-   MSAP-APP_CONFIG_DATA_RX.indication

-   MSAP-APP_CONFIG_DATA_RX.response (All response primitives have the same
    format, see section [Response
    Primitives](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#Response-Primitives))

Note:The MSAP-APP_CONFIG_DATA_RX service is available only in node role. With
sinks the configuration is set with MSAP-APP_CONFIG_DATA_WRITE service as
described above.

#### MSAP-APP_CONFIG_DATA_RX.indication

The MSAP-APP_CONFIG_DATA_RX.indication is issued by the stack when it receives
new configuration data from its neighbors. The
MSAP-APP_CONFIG_DATA_RX.indication frame is similar to
MSAP-APP_CONFIG_DATA_read.confirm frame (see section
[MSAP-APP_CONFIG_DATA_READ.confirm](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#MSAP-APP_CONFIG_DATA_READ.confirm)).
Only exception is that the *Result*-parameter is replaced with the
*indicationStatus* field (see section
[MSAP-INDICATION_POLL.confirm](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#MSAP-INDICATION_POLL.confirm)).

![](media/2b169d4da14a051491041fbcd6b4c807.png)

  
*Figure 19. MSAP-APP_CONFIG_DATA_RX.indication frame*

### MSAP-ATTRIBUTE_WRITE Service

The MSAP-ATTRIBUTE_WRITE service can be used by the application to write
run-time management attributes of the stack. The MSAP-ATTRIBUTE_WRITE service
includes the following primitives:

-   MSAP-ATTRIBUTE_WRITE.request

-   MSAP-ATTRIBUTE_WRITE.confirm

The MSAP attributes are specified in section [MSAP
Attributes](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#MSAP-Attributes).

#### MSAP-ATTRIBUTE_WRITE.request

The MSAP-ATTRIBUTE_WRITE.request is issued by the application when it wants to
set or change the MSAP attributes. The MSAP-ATTRIBUTE_WRITE.request frame is
depicted in Figure 20.

![](media/0154460cad735d0f1ffd3b1c56d29277.png)

  
*Figure 20. MSAP-ATTRIBUTE_WRITE.request frame*

  
The MSAP-ATTRIBUTE_WRITE.request frame fields are explained in Table 19.

  
*Table 19. MSAP-ATTRIBUTE_WRITE.request frame fields*

| **Field Name**    | **Size** | **Valid Values**                                                                                                                                                                 | **Description**                                                              |
|-------------------|----------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------|
| *AttributeID*     | 2        | Depends on the attribute. See section [MSAP Attributes](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#MSAP-Attributes) | The ID of the attribute that is written                                      |
| *AttributeLength* | 1        |                                                                                                                                                                                  | The length (in octets) of the attribute that is written                      |
| *AttributeValue*  | 1 – 16   |                                                                                                                                                                                  | The value that is written to the attribute specified by the set attribute ID |

#### MSAP-ATTRIBUTE_WRITE.confirm

The MSAP-ATTRIBUTE_WRITE.confirm is issued by the stack in response to the
MSAP-ATTRIBUTE_WRITE.request. The MSAP-ATTRIBUTE_WRITE.confirm frame is depicted
in Figure 21.

![](media/a564f6011cb678557861fb22b77219ee.png)

  
*Figure 21. MSAP-ATTRIBUTE_WRITE.confirm frame*

  
The MSAP-ATTRIBUTE_WRITE.confirm frame fields are explained in Table 20.

  
*Table 20. MSAP-ATTRIBUTE_WRITE.confirm frame fields*

| **Field Name** | **Size** | **Valid Values** | **Description**                                                                                                                                                                                                                                                                                                                                                                                       |
|----------------|----------|------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| *Result*       | 1        | 0 – 6            | The return result of the corresponding MSAP-ATTRIBUTE_WRITE.request. The different values are defined as follows:  0 = Success 1 = Failure: Unsupported attribute ID 2 = Failure: Stack in invalid state to write attribute 3 = Failure: Invalid attribute length 4 = Failure: Invalid attribute value  5 = Reserved  6 = Failure: Access denied (e.g. attribute read prevented by feature lock bits) |

### MSAP-ATTRIBUTE_READ Service

The MSAP-ATTRIBUTE_READ service can be used by the application layer to read
run-time management attributes from the stack. The MSAP-ATTRIBUTE_READ service
includes the following primitives:

-   MSAP-ATTRIBUTE_READ.request

-   MSAP-ATTRIBUTE_READ.confirm

#### MSAP-ATTRIBUTE_READ.request

The MSAP-ATTRIBUTE_READ.request is issued by the application when it wants to
read the MSAP attributes. The MSAP-ATTRIBUTE_READ.request frame is depicted in
Figure 22.

![](media/338f206f5e74e226799603e0af1c395b.png)

  
*Figure 22. MSAP-ATTRIBUTE_READ.request frame*

  
The MSAP-ATTRIBUTE_READ.request frame fields are explained in Table 21.

  
*Table 21. MSAP-ATTRIBUTE_READ.request frame fields*

| **Field Name** | **Size** | **Valid Values**                                                                                                                                                                 | **Description**                      |
|----------------|----------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------|
| *AttributeID*  | 2        | Depends on the attribute. See section [MSAP Attributes](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#MSAP-Attributes) | The ID of the attribute that is read |

#### MSAP-ATTRIBUTE_READ.confirm

The MSAP- ATTRIBUTE_READ.confirm is issued by the stack in response to the
MSAP-ATTRIBUTE_READ.request. The MSAP-ATTRIBUTE_READ.confirm frame is depicted
in Figure 23.

![](media/97ff3ef5e5d4725ba5ff5588bf90dac9.png)

  
*Figure 23. MSAP-ATTRIBUTE_READ.confirm frame*

  
The MSAP-ATTRIBUTE_READ.confirm frame fields are explained in Table 22.

  
*Table 22. MSAP-ATTRIBUTE_READ.confirm frame fields*

| **Field Name**    | **Size** | **Valid Values**                                                                                                                                                                  | **Description**                                                                                                                                                                                                                                                                                                                                                                                                                                               |
|-------------------|----------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| *Result*          | 1        | 0 – 6                                                                                                                                                                             | The return result of the corresponding MSAP-ATTRIBUTE_READ.request. The different values are defined as follows:  0 = Success 1 = Failure: Unsupported attribute ID 2 = Failure: Stack in invalid state to read attribute 4 = Failure: Invalid attribute value or attribute value not yet set 5 = Failure: Write-only attribute (e.g. the encryption and authentication keys) 6 = Failure: Access denied (e.g. attribute read prevented by feature lock bits) |
| *AttributeID*     | 2        | Depends on the attribute. See section [MSAP Attributes](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#MSAP-Attributes). | The ID of the attribute that is read                                                                                                                                                                                                                                                                                                                                                                                                                          |
| *AttributeLength* | 1        |                                                                                                                                                                                   | The length (in octets) of the attribute that is read                                                                                                                                                                                                                                                                                                                                                                                                          |
| *AttributeValue*  | 1 – 16   |                                                                                                                                                                                   | The value of the read attribute specified by the set attribute ID. This value of the attribute is only present if *Result* is 0 (Success).                                                                                                                                                                                                                                                                                                                    |

### MSAP-GET_NBORS Service

This service can be used to tell the status of a node's neighbors. This
information may be used for various purposes, for example to estimate where a
node is located.

#### MSAP-GET_NBORS.request

MSAP-GET_NBORS.request is issued by the application layer to query information
about neighboring nodes. The MSAP-GET_NBORS.request frame is depicted in Figure
24. It contains no payload.

![](media/94f4b6fe71625d4ffd7b18b4848f3f88.png)

  
*Figure 24. MSAP-GET_NBORS.request frame*

#### MSAP-GET_NBORS.confirm

MSAP-GET_NBORS.confirm is issued by the stack as a response to the
MSAP-GET_NBORS.request. The MSAP-GET_NBORS.confirm frame is depicted in Figure
25 and Figure 26.  
Information for a maximum of eight neighbors is contained within one
MSAP-GET_NBORS.confirm frame. The frame size does not change. If number of
neighbors is less than eight, remaining data in the frame is undefined. If
access is denied (see section
[cFeatureLockBits](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#cFeatureLockBits))
a block of zeros is returned.

  


![](media/74502598b5d5f9d0b2921cd04d40c349.png)

  
*Figure 25. MSAP-GET_NBORS.confirm frame*

  
The neighbor info frame contains information for a maximum of eight neighbors.
This information is depicted in Figure 26.

![](media/f988ace2f845eb482ae4ecd1ba69c4eb.png)

  
*Figure 26. MSAP-GET_NBORS.confirm neighbor info*

  
The MSAP-GET_NBORS.confirm frame fields are explained in Table 23.

  
*Table 23. MSAP-GET_NBORS.confirm frame fields*

| **Field Name**      | **Size** | **Valid Values**                    | **Description**                                                                                                                                                 |
|---------------------|----------|-------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| *NumberOfNeighbors* | 1        | 0 – 8                               | Number of neighbors' information returned. 0 if there are no neighbors.                                                                                         |
| *NeighborAddress*   | 4        | 0 – 4294967295                      | Address of the neighbor node                                                                                                                                    |
| *LinkReliability*   | 1        | 0 – 255                             | Link reliability to the neighboring node. Scaled so that 0 = 0 %, 255 = 100 %                                                                                   |
| *NormalizedRSSI*    | 1        | 0 – 255                             | Received signal strength, compensated with transmission power. Larger value means better the signal.  0: No signal 1: Signal heard barely\>50: Good signal      |
| *Cost*              | 1        | 1 – 255                             | Route cost to the sink via this neighbor. Value 255 indicates that a neighbor has no route to a sink.                                                           |
| *Channel*           | 1        | 1 – CSAP attribute *cChannelLimits* | Radio channel used by the neighbor                                                                                                                              |
| *NeighborType*      | 1        | 0 – 2                               | Type of neighbor  0: Neighbor is next hop cluster, i.e. used as a route to sink 1: Neighbor is a member of this node 2: Neighbor is a cluster from network scan |
| *TxPower*           | 1        | 0 – X                               | Power level used for transmission  0: Lowest power  X: Highest power (depending on the stack profile)                                                           |
| *RxPower*           | 1        | 0 – X                               | Received power level  0: Lowest power  X: Highest power (depending on the stack profile)                                                                        |
| *LastUpdate*        | 2        | 0 – 65535                           | Amount of seconds since these values were last updated                                                                                                          |

### MSAP-SCAN_NBORS Service

This service can be used by the application to get fresh information about
neighbors. Application can trigger to measurement all neighbors and once the
measurement is done, application is informed it over API. The MSAP-SCAN_NBORS
service includes the following primitives:

-   MSAP-SCAN_NBORS.request

-   MSAP-SCAN_NBORS.confirm

-   MSAP-SCAN_NBORS.indication

-   MSAP-SCAN_NBORS.response (All response primitives have the same format, see
    section [Response
    Primitives](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#Response-Primitives))

#### MSAP-SCAN_NBORS.request

MSAP-SCAN_NBORS.request is issued by the application when it starts to
measurement all neighbors. The MSAP-SCAN_NBORS.request frame is depicted in
Figure 24. It contains no payload.

![](media/94f4b6fe71625d4ffd7b18b4848f3f88.png)

  
*Figure 27. MSAP-SCAN_NBORS.request frame*

#### MSAP-SCAN_NBORS.confirm

This confirm tells result which is always success. After application has asked
to scan neighbors so stack code use signal to handle it.

![](media/02a83d958d64257329639bc881598c46.png)

  
*Figure 28. MSAP-SCAN_NBORS.confirm frame*

  
The MSAP-SCAN_NBORS.confirm frame fields are explained in following table.

  
*Table 24. MSAP_SCAN_NBORS.confirm frame fields*

| **Field Name** | **Size** | **Valid Values** | **Description**                                                                                                                                                                                                                                                                                                            |
|----------------|----------|------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| *Result*       | 1        | 0 – 2            | The return result of the corresponding MSAP-SCAN_NBORS.confirm.  0 = Success  1 = Failure: Stack in invalid state, i.e. not running  2 = Failure: Access denied (see section [cFeatureLockBits](https://wirepas.atlassian.net/wiki/spaces/CD/pages/175047279/Wirepas+Mesh+Dual-MCU+API+Reference+Manual#cFeatureLockBits)) |
