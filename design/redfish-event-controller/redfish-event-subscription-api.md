# redfish-event-subscription-api

## Status

provisional

## Summary

This design describes an API for allowing applications to subscribe to
event notifications from baseboard management controllers that support
the Redfish protocol. The API can be used for hardware monitoring by
the platform or by applications running on the platform.

## Motivation

The Redfish specification describes an API for subscribing to "events"
or "alerts" using a webhook. Events are usually generated when the
configuration of the system changes, or when telemetry data such as
temperature or fan speed falls outside of a desired range. We want to
expose these events in clusters using metal3, without giving the
application that will use the event data credentials to access the BMC
directly.

### Goals

* Describe an API for managing notification subscriptions for
  Redfish-based management controllers

### Non-Goals

* This design will not consider systems that do not use the Redfish
  protocol.
* This design will not consider the design of the application that
  receives the event notifications, beyond the fact that it must use
  an https endpoint that supports the POST method.

## Proposal

A new API for describing an `EventSubscription` will be created. The
first version will be limited to the standard parameters to the
Redfish API, and may be extended later to support the `oem`
argument. A new standalone controller will be created to manage the
new API.

The new CRD name is `EventSubscription` in the `redfish.metal3.io`
group, and it will have the following fields:

`spec.hostRef`

  An `EventSubscription` is linked to a management controller through
  an object reference to the `BareMetalHost` resource. This means that
  permission to create and delete subscriptions can be independent of
  permissions to change host resources. The controller for the
  subscription API will need permission to read the host resource and
  its secret, to get the location and credentials of the host.

`spec.destinationURL`

  Events are POSTed to a URL given to the management controller. This
  field is passed through to the controller without modification.

`spec.eventTypes`

  Redfish defines a set of event types so a subscription can be
  filtered. This field will be a list of values based on an enum
  derived from the Redfish specification and vendor-specific
  implementations.

`status.errorMessage`

  If any errors occur when configuring the subscription on the
  management controller, the message is saved to this field so the API
  user can debug the problem.

`status.ODataID`

  When a subscription is successfully created on a management
  controller, it is assigned a URI. This value is reported through the
  Redfish API as the `ODataID`, so this field is named the same.

```yaml
apiVersion: redfish.metal3.io/v1alpha1
kind: EventSubscription
metadata:
  name: test
spec:
  hostRef:
    name: my-baremetalhost-name
  destinationURL: https://10.8.1.133:9090/test?var=val
  eventTypes:
    - Alert
    - ResourceAdded
    - ResourceRemoved
    - ResourceUpdated
    - StatusChange
```

### User Stories

1. As a cluster administrator, I want to be notified if my hardware
   reports any alerts based on telemetry.
2. As an application developer, I want to be notified if the hardware
   I am running on reports any faults or alerts so I can react quickly
   and finish any tasks currently in process cleanly or stop accepting
   additional tasks.

## Design Details

The new API should be managed by a new controller, tentatively named
the `redfish-event-controller`. The controller can be deployed
independently of the rest of metal3, except that it uses the
`BareMetalHost` API defined by the `baremetal-operator`.

### Implementation Details/Notes/Constraints

The notification payloads are JSON documents POSTED to the destination
URL for each subscription.  Here is an example for an alert triggered via the test UI on a Dell iDrac 8:

```json
{
  "Context": "metal3:90db7bf4-4b0d-4d17-966c-f978a0083c6b:openshift-machine-api/test:Z1LkpKyk+ztTGN07EaJBXS6yVcjD1KinJy/jkOzvppY=",
  "EventId": "2241",
  "EventTimestamp": "2020-11-14T17:15:06-0500",
  "EventType": "Alert",
  "MemberId": "ba88f0ce-26c6-11eb-a058-588a5afa1cc4",
  "Message": "CPU 1 has a thermal trip (over-temperature) event.",
  "MessageArgs": [
    "1"
  ],
  "MessageArgs@odata.count": 1,
  "MessageId": "CPU0001",
  "Severity": "Critical"
}
```

and another example from a login event on the same BMC:

```json
{
  "Context": "metal3:90db7bf4-4b0d-4d17-966c-f978a0083c6b:openshift-machine-api/test:Z1LkpKyk+ztTGN07EaJBXS6yVcjD1KinJy/jkOzvppY=",
  "EventId": "8491",
  "EventTimestamp": "2020-11-14T17:15:39-0500",
  "EventType": "Alert",
  "MemberId": "ba88f0ce-26c6-11eb-a058-588a5afa1cc4",
  "Message": "Successfully logged in using OSonOS, from 10.8.1.133 and REDFISH.",
  "MessageArgs": [
    "OSonOS",
    "10.8.1.133",
    "REDFISH"
  ],
  "MessageArgs@odata.count": 3,
  "MessageId": "USR0030",
  "OriginOfCondition": "iDRAC.Embedded.1",
  "Severity": "Informational"
}
```

The `Context` field in each case is generated by the controller to
uniquely identify the subscription.

Redfish does not support editing subscriptions, so the controller will
have to manage changes in another way. The proof-of-concept
implementation linked below encodes a signature for the input
parameters in the resource in the `Context` of the Redfish event
subscription, so it can be compared with the latest version of the
`EventSubscription` resource during reconciliation.

The `oem` fields of the Redfish API are ignored, for now. Those fields
can be used to configure vendor-specific behavior for things like how
often to retry posting messages if there is an error and how long to
wait between retries. If those values need to be exposed, we will use
another design document to work out the API changes.

### Risks and Mitigations

There is some risk that making it easy to subscribe to hardware events
will cause cluster administrators to over-use the feature, and tax the
notification system in the management controller. To mitigate this, we
can recommend that users attach the events to a message bus system for
broadcast by multiple consumers, to minimize the number of URLs
registered with each BMC.

There is no apparent limit to the number of subscriptions that can be
supported, although there will be a practical limit based on the
ability to store the subscription settings in the firmware. If a user
does hit the limit, the controller will report an error registering
the new subscription.

Exposing hardware events directly to user workloads may also present a
security risk. This can be mitigated by using a separate resource type
with separate RBAC, as described in this design.

The current implementation has been tested only with Dell hardware. We
will need to find other types of hardware to ensure that the basic
Redfish behavior is compatible.

### Work Items

1. Create the basic controller done by importing the repository
   containing the proof-of-concept implementation into `metal3-io` on
   GitHub.
2. Recruit a few more maintainers.
3. Set up CI for the new repository.
4. Document the new API.
5. Update the proof-of-concept implementation to handle deleting
   subscriptions.

### Dependencies

* The new API and controller depend on the `baremetal-operator` and
  `BareMetalHost` API.
* The new controller uses the
  [gofish](https://github.com/stmcginnis/gofish) library to
  communicate directly with the management controller.

### Test Plan

* Unit tests
* Automated tests using a mock server

### Upgrade / Downgrade Strategy

N/A

### Version Skew Strategy

N/A

## Drawbacks

Exposing hardware events directly to user workloads may also present a
security risk. This can be mitigated by using a separate resource type
with separate RBAC, as described in this design.

## Alternatives

### Extend the BareMetalHost API

We could add event subscriptions to the BareMetalHost API. However,
this would require granting full access to the host for any
consumer. It would also make it more difficult to determine when a
subscription was added or removed, so the BMC settings could be kept
up to date.

### Use Ironic to manage subscriptions

Ironic does not currently have an API for managing event
subscriptions, but one could be added.

Using Ironic would introduce an extra layer of code between the
kubernetes API and the Redfish API, which might be useful in the
future if other protocols are supported. On the other hand, those
other protocols are likely to have different inputs anyway, so trying
to abstract the API at this point is unlikely to be very successful.

Using Ironic would also place the vendor-specific logic for managing
subscriptions in a single place. There does not seem to be very much
vendor-specific logic for this API, unlike the more complex APIs for
provisioning or managing virtual media. In particular, there is no
"workflow" needed for these simple APIs.

## References

* [Presentation on Redfish events from HPE
  engineer](https://www.dmtf.org/sites/default/files/Redfish School -
  Events.pdf)
* [blog post about Redfish events from
  HPE](https://developer.hpe.com/blog/the-redfish-event-service)
* [Dell documentation for
  subscriptions](https://www.dell.com/support/manuals/en-us/idrac9-lifecycle-controller-v4.x-series/idrac9_4.00.00.00_redfishapiguide_pub/subscription?guid=guid-fb1b10a1-c371-456f-85aa-3a1415da5b58&lang=en-us)
* [DMTF/Redfish-event-listener example
  code](https://github.com/DMTF/Redfish-Event-Listener)
* [Lenovo docs that explain the
  fields](https://sysmgt.lenovofiles.com/help/index.jsp?topic=%2Fcom.lenovo.systems.management.xcc.restapi.doc%2Fcreate_a_subscription_post.html)
* [DMTF
  documentation](http://redfish.dmtf.org/schemas/DSP0266_1.8.0.html#eventing)
* [proof-of-concept
  implementation](https://github.com/dhellmann/redfish-event-controller)
