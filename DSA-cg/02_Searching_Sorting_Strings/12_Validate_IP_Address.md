# Validate IP Address

**Difficulty:** Medium  
**Topics:** Parsing, validation, IPv4, IPv6

## 1. Problem statement

Classify a string as valid IPv4, valid IPv6, or neither. IPv4 has four decimal octets (0–255), with no leading zero unless the octet is exactly `"0"`. IPv6 has eight hexadecimal groups of one to four digits in this simplified full-form validator; compressed `::` notation is not accepted by this version.

## 2. Approaches

### Brute-force idea

Split on dots or colons and check each segment. Avoid permissive conversions alone: `int("01")` succeeds but leading zero is invalid in this contract.

### Explicit validation

```python
import re

def valid_ipv4(ip):
    parts = ip.split(".")
    if len(parts) != 4:
        return False
    for part in parts:
        if not part or (len(part) > 1 and part[0] == "0"):
            return False
        if not part.isdigit():
            return False
        if not 0 <= int(part) <= 255:
            return False
    return True

def valid_ipv6_full(ip):
    parts = ip.split(":")
    if len(parts) != 8:
        return False
    return all(1 <= len(p) <= 4 and re.fullmatch(r"[0-9a-fA-F]+", p)
               for p in parts)

def valid_ip(ip):
    if valid_ipv4(ip):
        return "IPv4"
    if valid_ipv6_full(ip):
        return "IPv6"
    return "Neither"
```
O(L) time and O(L) space for split segments, where L is input length.

### Production-grade standard library

```python
import ipaddress

def classify_ip_strict(ip):
    try:
        address = ipaddress.ip_address(ip)
        return "IPv4" if address.version == 4 else "IPv6"
    except ValueError:
        return "Neither"
```
Use this when standards-compliant parsing is expected. State the exact accepted syntax.

## 3. Dry run

`"172.16.254.1"` splits into four segments. Each is decimal, has no forbidden leading zero, and lies in 0..255: IPv4. `"256.1.1.1"` fails the range check. `"2001:db8:0:0:0:0:2:1"` has eight valid hexadecimal groups: IPv6 under this full-form rule.

## 4. Interviewer–candidate discussion

**Interviewer:** Does your IPv6 validator support `::` compression?

**Candidate:** The explicit function does not. Supporting it requires rules for at most one compression marker and expansion to exactly eight groups. For production, I prefer Python's `ipaddress` module.

**Interviewer:** What about whitespace?

**Candidate:** I would not silently strip it unless the input contract permits surrounding whitespace.

## 5. Edge cases and mistakes

Leading zeros, empty segments, too many/few groups, invalid hex characters, IPv4 out of range, multiple `::`, embedded IPv4 notation, whitespace. Do not present a simplified validator as fully RFC-compliant.

## 6. Tests

```python
assert valid_ip("172.16.254.1") == "IPv4"
assert valid_ip("256.1.1.1") == "Neither"
assert valid_ip("01.1.1.1") == "Neither"
assert valid_ip("2001:db8:0:0:0:0:2:1") == "IPv6"
assert valid_ip("2001:db8::1") == "Neither"  # compressed form excluded here
```
Follow-up: implement compressed IPv6 validation or explain why a standard parser is safer.
