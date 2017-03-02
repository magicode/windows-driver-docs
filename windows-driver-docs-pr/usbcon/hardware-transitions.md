---
Description: 'This topic describes hardware transitions to U1 and U2.'
MS-HAID: 'buses.hardware\_transitions'
MSHAttr:
- 'PreferredSiteName:MSDN'
- 'PreferredLib:/library/windows/hardware'
title: Hardware transitions
---

# Hardware transitions


This topic describes hardware transitions to U1 and U2.

After the initial setup by the software, the hardware transitions to U1 and u2 autonomously without further intervention from the software.

A link is in working state (U0) as long as it is actively transferring packets. The link is considered to be idle when no packets are being transmitted. In idle state, any link partner can initiate a transition to U1 or U2. The other link partner can choose to accept or reject the transition. If the link partner accepts the transition, the link moves to that U state. If it rejects the transition, the link remains in U0.

## DS port-initiated transitions


A DS port implements a timer mechanism that tracks inactivity on the port. The timer gets reset whenever that port sends or receives a packet. The timer also gets reset when the software programs new timeout values. If the software has programmed the DS port to initiate only U1 or U2 transitions, the DS port starts the timer when the link first enters U0. The timer value is based on the U1 (or U2) timeout value that was programmed by the software. If the link is in U0 when the timer expires, the DS port initiates the U1 (or U2) transition.

The US port link partner can choose to reject the transition if the device knows that the transition can affect the device’s ability to meet the performance or latency requirements. For example, if the device has sent an ERDY notification and is expecting a transfer request from the host, then the device might reject U1 or U2 state transitions in the meantime.

If the software has programmed the DS port to initiate both U1 and U2 transitions, the DS port first initiates the U1 transition based on the timer (described earlier in this section). The transition from U1 to U2 is described in this topic in **Direct Transition from U1 to U2**.

If a link is in U1 or U2, a DS port can bring the port back into U0 anytime it receives traffic for the device attached to the port.

## Device (US port)-initiated transitions


A device can choose to initiate a transition from U0 to U1 or U0 to U2, as long as the capability is enabled by the software. If the device transitions a link to U1, the link can transition to U2 directly based on the U2 timer of the DS port (described in [Direct Transition from U1 to U2](#u1tou2)). However, if the U2 timer is not set, the device cannot initiate a direct transition from U1 to U2 on its own. In that case, the device must bring the link back to U0 before initiating the transition to U2.

While deciding when to initiate those transitions, a device should consider its exit latencies and performance requirements. To help the device make informed decisions about how aggressively it can initiate the transitions, software also provides various exit latency values as described earlier in this document in [Initial setup by software](initial-setup-by-software.md).

If the link is in U1 or U2, a US port can bring the port back into U0 at any time. Typically, the US port initiates the transition to U0 when it knows that it is about to send any packets to the host or if it is anticipating a packet from the host.

## Advantages of device-initiated LPM


The timer values that are set by the software for DS ports are based on general heuristics. While choosing those timer values, software ensures that the device performance is not impaired. In order to maintain the device’s performance, software cannot choose values that are too small. Because DS port- initiated transitions are based on the timers and do not take into account the exact state of the device, this mechanism cannot take advantage of all the possible opportunities of sending the device to U1 or U2 state.

The device, on the other hand, has accurate knowledge about its characteristics and current state. Therefore, it can make an intelligent guess about when the next transfer is going to take place. Based on that information, the device can (and should) choose to actively initiate these transitions without significantly affecting the performance.

For example, the device has sent a NRDY notification on one of its endpoints and knows that there will not be traffic for a while. In that case, the device can immediately initiate a transition to U1 or U2. Just before sending the ERDY notification, the device can bring the link back to U0 in preparation for sending that data. For details about this process, see section C.3.1 of the USB 3.0 specification.

## <a href="" id="u1tou2"></a>Direct transition from U1 to U2


If the link is in U1, it is possible that the link can directly transition to U2 without entering U0 in between. That can occur regardless of which link partner initiated the transition to U1. However, the U1 to U2 transition can occur only if the U2 timeout on the DS port of the link is set to a value between 01H-FEH.

The [Initial setup by software](initial-setup-by-software.md) section describes an additional step that allows the DS port to communicate the timeout value to its link partner. After the link has entered U1, both link partners start a timer using the timeout value set according to the U2 timeout value of the DS port. If the timer is not reset due to traffic and expires, both link partners silently transition to U2 without any explicit communication between them.

## Transitions from U1 or U2 to U3


Transitions to U1 or U2 are initiated in the hardware autonomously but the transition to U3 is initiated by the software. Because U3 transition is initiated only after a period of inactivity, it is quite likely that the link was in U1 or U2 (rather than U0) before the transition.

The USB 3.0 specification does not define direct transitions from U1 or U2 to U3. The parent hub or controller is responsible for automatically transitioning the link to U0 and then transitioning it to U3.

## U1 or U2 transitions for hubs


The USB 3.0 specification provides specific guidelines for hubs about when to initiate U state transitions on its US port. If all the DS ports are in link state U1 or lower, the hub should initiate a U1 transition on its US port, assuming that the software enabled the hub to initiate the U1 transition.

Similarly, if all DS ports are in link state U2 or lower, the hub should initiate a U2 transition on its US port, assuming that the software enabled the hub to initiate the U2 transition.

**Note**  
If there is no device attached to a DS port, the port’s state is Rx.Detect, which is lower than U2. So if there are no devices attached, the hub should send its US port to U2. Also, if all the DS ports were initially in U1 or lower and they transition to U2 or lower, the hub should transition the US port from U1 to U2. Because that transition is not based on U2 activity timer, the hub must bring its US port to U0 and then send it to U2.

 

## <a href="" id="packet-def"></a>Packet deferring


The USB 3.0 specification describes a mechanism known as packet deferring (see section C.1.2.2). The mechanism is used to minimize the effect of LPM on bus utilization.

If a host sends a transfer request to a device, whose upstream link is in U1 or U2, the host could end up wasting bus bandwidth by waiting for the link to come back to U0 and then for the device to respond. To avoid that wait, the parent hub responds on behalf of the device by sending a deferred packet header back to the host. The host processes the deferred packet header in a manner similar to NRDY, and is then free to initiate transfers with other endpoints. In parallel, the hub initiates a U0 transition on the link and then informs the device about the deferred packet. The device then sends ERDY to the host to indicate that the device is now ready for the transfer. The host can then reschedule the transfer to the device.

An important responsibility of the device is that after sending ERDY, the device is responsible for keeping the link in U0 until the host sends a response to ERDY or until the **tERDYTimeout** (500 milliseconds) time elapses. During that time, the device must not initiate a U1 or U2 transition and should also reject any transition initiated by its link partner.

 

 

[Send comments about this topic to Microsoft](mailto:wsddocfb@microsoft.com?subject=Documentation%20feedback%20%5Busbcon\buses%5D:%20Hardware%20transitions%20%20RELEASE:%20%281/26/2017%29&body=%0A%0APRIVACY%20STATEMENT%0A%0AWe%20use%20your%20feedback%20to%20improve%20the%20documentation.%20We%20don't%20use%20your%20email%20address%20for%20any%20other%20purpose,%20and%20we'll%20remove%20your%20email%20address%20from%20our%20system%20after%20the%20issue%20that%20you're%20reporting%20is%20fixed.%20While%20we're%20working%20to%20fix%20this%20issue,%20we%20might%20send%20you%20an%20email%20message%20to%20ask%20for%20more%20info.%20Later,%20we%20might%20also%20send%20you%20an%20email%20message%20to%20let%20you%20know%20that%20we've%20addressed%20your%20feedback.%0A%0AFor%20more%20info%20about%20Microsoft's%20privacy%20policy,%20see%20http://privacy.microsoft.com/default.aspx. "Send comments about this topic to Microsoft")


