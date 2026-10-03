# Network Troubleshooting Tickets

This document records three simulated support incidents completed during the Small-Business Network Troubleshooting Lab. Each ticket follows the same professional workflow: reproduce the symptom, gather evidence, identify the root cause, apply the smallest appropriate fix, and verify service restoration.

## Ticket NT-001: HR-PC2 Cannot Access the Network

### User report

> HR-PC2 cannot connect to department resources or the company server.

### Business impact

The affected HR employee cannot reach the default gateway, internal server, DNS, or intranet.

### Initial symptoms

- The Ethernet cable and switch link remained active.
- HR-PC2 retained an address in `192.168.30.0/24`.
- The configured default gateway was `192.168.30.1`.
- Pings to the HR gateway and server failed.

### Diagnostic actions

1. Ran `ipconfig` on HR-PC2 to verify its IPv4 address, mask, and gateway.
2. Confirmed the physical link was active.
3. Ran `show vlan brief` on `SW1-ACCESS`.
4. Discovered Fa0/6 listed under Sales VLAN 20 instead of HR VLAN 30.

### Root cause

The switch access port connected to HR-PC2 was assigned to the wrong VLAN. The computer had a valid HR address but was physically placed inside the Sales broadcast domain.

### Resolution

```text
configure terminal
interface fastEthernet0/6
switchport access vlan 30
end
```

### Verification

- `show vlan brief` displayed Fa0/6 under VLAN 30.
- HR-PC2 successfully pinged `192.168.30.1`.
- HR-PC2 successfully pinged `192.168.40.10`.

### Evidence

- [Failed ping](../Screenshots/ticket-01-failed-ping.png)
- [Incorrect VLAN assignment](../Screenshots/ticket-01-wrong-vlan.png)
- [Successful ping after repair](../Screenshots/ticket-01-fixed-ping.png)

### Lesson learned

A correct IP configuration does not guarantee connectivity. The switch port must also belong to the VLAN that matches the endpoint's IP subnet.

---

## Ticket NT-002: Intranet Fails by Name

### User report

> The company intranet will not open using its normal name.

### Business impact

Employees cannot reach the internal portal through the documented URL, even though the web server remains online.

### Initial symptoms

- `ping 192.168.40.10` succeeded.
- `http://192.168.40.10` loaded successfully.
- `ping intranet.corp.voscrez.test` failed to resolve.
- `http://intranet.corp.voscrez.test` failed.

### Diagnostic actions

1. Tested the server by IP address to verify Layer 3 connectivity.
2. Opened the webpage by IP to verify the HTTP service.
3. Tested the DNS hostname separately.
4. Verified that DHCP had provided `192.168.40.10` as the client DNS server.
5. Inspected the DNS service on `CORP-SRV1` and found it stopped.

### Root cause

The DNS service on `CORP-SRV1` was turned off. Routing and HTTP remained functional, but clients could not translate the intranet hostname into the server IP address.

### Resolution

Restarted DNS on `CORP-SRV1` and confirmed the following A record remained present:

```text
intranet.corp.voscrez.test -> 192.168.40.10
```

### Verification

- The hostname resolved to `192.168.40.10`.
- The hostname responded to ping.
- The intranet loaded successfully through its DNS name.

### Evidence

- [DNS failure evidence](../Screenshots/ticket-02-dns-failure.png)
- [Successful DNS and website test](../Screenshots/ticket-02-dns-fixed.png)

### Lesson learned

When an application works by IP address but fails by hostname, basic routing is usually working. DNS should be investigated before making unrelated network changes.

---

## Ticket NT-003: IT-PC2 Has No Network Connection

### User report

> IT-PC2 suddenly lost all network access.

### Business impact

The affected IT employee cannot reach the gateway, server, DNS, or other company systems.

### Initial symptoms

- The Packet Tracer link indicator turned red.
- IT-PC2 could not ping `192.168.10.1`.
- The problem affected only the endpoint connected to Fa0/2.

### Diagnostic actions

1. Checked the visible link status.
2. Ran `show interfaces status` on `SW1-ACCESS`.
3. Found Fa0/2 in the `disabled` state.
4. Ran `show interfaces fastEthernet0/2` and confirmed it was administratively down.

### Root cause

The Fa0/2 switch interface had been manually shut down. This disabled the endpoint's network connection even though its IP settings were correct.

### Resolution

```text
configure terminal
interface fastEthernet0/2
no shutdown
end
```

### Verification

- The link indicator returned to green.
- Fa0/2 returned to the connected state.
- IT-PC2 successfully pinged `192.168.10.1`.
- IT-PC2 successfully pinged `192.168.40.10`.

### Evidence

- [Disabled interface](../Screenshots/ticket-03-disabled-port.png)
- [Successful connectivity after repair](../Screenshots/ticket-03-fixed-port.png)

### Lesson learned

A red or inactive link suggests a Layer 1 or interface-state problem. Checking interface status can distinguish an administratively disabled port from an IP, routing, or application problem.

---

## Overall Troubleshooting Takeaway

The three incidents occurred at different parts of the network:

| Ticket | Area | Failure |
|---|---|---|
| NT-001 | Layer 2 segmentation | Incorrect VLAN assignment |
| NT-002 | Application service | DNS service stopped |
| NT-003 | Physical/interface state | Switch port shut down |

Testing one layer at a time prevented random configuration changes and made each root cause easier to isolate.
