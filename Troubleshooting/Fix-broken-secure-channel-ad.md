# Troubleshooting a Broken Secure Channel Between Client and Domain Controller

## Summary

A domain-joined Windows client (`CLIENT-PC`) in the `example.local` domain was unable to apply Group Policy and was failing to authenticate against the domain controller (`DC01.example.local`, `10.0.0.10`). The root cause turned out to be a **broken secure channel** (computer account trust) between the client and the domain — not a DNS or network connectivity issue, even though those were the first suspects.

This document walks through the diagnostic steps taken, in order, and the resolution.

---

## Symptoms

Running `gpupdate /force` on the client returned:

```
Computer policy could not be updated successfully. The following errors were encountered:

The processing of Group Policy failed because of lack of network connectivity to a domain controller.
This may be a transient condition. A success message would be generated once the machine gets connected
to the domain controller and Group Policy has successfully processed. If you do not see a success message
for several hours, then contact your administrator.

User Policy could not be updated successfully. The following errors were encountered:
[same message]
```

---

## Investigation

### 1. Checked DNS resolution to the DC

```cmd
ping dc01
```

This initially resolved to a suspicious address (IPv6 loopback `::1`), which turned out to be an unrelated legacy/stale DNS record from a different naming convention in the environment — not the actual issue for this domain. Worth ruling out early, since a bad AAAA/A record pointing to loopback would fully explain connectivity failures on its own.

### 2. Confirmed the actual domain controller via secure channel query

```cmd
nltest /sc_query:example.local
```

Result:

```
Trusted DC Connection Status Status = 1311 0x51f ERROR_NO_LOGON_SERVERS
```

This error means the client cannot find **any** usable domain controller to authenticate against — but it doesn't distinguish between a DNS problem, a network problem, or a trust problem.

### 3. Verified DNS could locate a DC via SRV records

```cmd
nslookup -type=srv _ldap._tcp.dc._msdcs.example.local
```

Result: successfully resolved to `dc01.example.local` at `10.0.0.10`. **DNS discovery was working fine.**

### 4. Verified DC discovery end-to-end

```cmd
nltest /dsgetdc:example.local
```

Result: fully successful — returned the DC name, IP, domain GUID, and a full set of healthy flags (`PDC GC DS LDAP KDC TIMESERV GTIMESERV WRITABLE DNS_DC DNS_DOMAIN DNS_FOREST ...`).

This confirmed the domain controller itself was healthy and reachable. So DNS, network path, and the DC's own services were **not** the problem.

### 5. Tested the client's secure channel directly

```powershell
Test-ComputerSecureChannel -Verbose
```

Result:

```
False
VERBOSE: The secure channel between the local computer and the domain example.local is broken.
```

**This was the root cause.** The client's machine account trust with the domain (its computer account password) was out of sync with what AD had on record.

---

## Attempted Fix #1: Repair the secure channel in place

```powershell
$cred = Get-Credential
Test-ComputerSecureChannel -Repair -Credential $cred -Verbose
```

Result: repair failed.

```
VERBOSE: The attempt to repair the secure channel between the local computer and the domain example.local has failed.
```

Possible reasons this can fail even with valid domain admin credentials:
- The AD computer object is disabled or missing entirely
- The credentials used don't have "Reset Password" / "Reset Computer Account" rights on that specific object
- The object exists but the state is corrupted enough that an in-place repair can't reconcile it

---

## Resolution: Unjoin and rejoin the domain

Since the in-place repair failed, the reliable fix was to remove the machine from the domain and rejoin it, generating a fresh computer account trust.

### Steps taken

1. Logged in locally using a **local administrator account** (not a domain account, since domain auth was unreliable).
2. Removed the machine from the domain and moved it to a workgroup:
   ```powershell
   Remove-Computer -WorkgroupName "WORKGROUP" -Force -Restart
   ```
   (Using `Remove-Computer` with local admin rights avoided needing to authenticate against the already-broken domain trust.)
3. Rebooted.
4. Rejoined the domain via **System Properties → Computer Name → Change**, supplying domain admin credentials.
5. Rebooted again.
6. Verified the fix:
   ```powershell
   Test-ComputerSecureChannel -Verbose
   ```
   Returned `True`.
7. Confirmed Group Policy processing was restored:
   ```cmd
   gpupdate /force
   ```
   Completed successfully with no errors.

---

## Root Cause

The client's local computer account password had fallen out of sync with the copy Active Directory held for it. This is commonly caused by:
- Restoring a VM/machine from an old snapshot or backup (the machine "goes back in time" to a password AD has already rotated past)
- The machine being offline for longer than the default computer password change interval (30 days) combined with some other disruption
- The AD computer object being deleted, disabled, or reset on the domain side without the client being rejoined

Because the DC itself was fully healthy and DNS was resolving correctly, none of the network-layer checks (`ping`, `nslookup`, `nltest /dsgetdc`) revealed the issue — only `Test-ComputerSecureChannel` pointed directly at the trust relationship being broken.

---

## Key Takeaways / Lessons Learned

- `ERROR_NO_LOGON_SERVERS (1311)` does **not** necessarily mean a DNS or network problem — it can also mean the client simply can't authenticate due to a broken trust, even when it can fully see and discover the DC.
- Always test in this order to isolate the layer that's actually broken:
  1. DNS resolution (`nslookup`)
  2. DC discovery (`nltest /dsgetdc`)
  3. Port-level reachability (`Test-NetConnection <DC IP> -Port 389/445/88`)
  4. Secure channel / trust (`Test-ComputerSecureChannel`)
- `Test-ComputerSecureChannel -Repair` is worth trying first since it's non-disruptive, but it can fail silently or explicitly if the AD computer object itself is in a bad state.
- Unjoin/rejoin is a reliable fallback that resolves almost any broken secure channel, at the cost of a couple of reboots and needing local admin access.
- If BitLocker is enabled on the affected machine, have the recovery key on hand before unjoining — domain leave/rejoin can occasionally trigger a recovery prompt.

---

## Commands Reference

| Purpose | Command |
|---|---|
| Force GPO update | `gpupdate /force` |
| Query secure channel status | `nltest /sc_query:<domain>` |
| Query SRV records for DC discovery | `nslookup -type=srv _ldap._tcp.dc._msdcs.<domain>` |
| Full DC discovery | `nltest /dsgetdc:<domain>` |
| Test secure channel | `Test-ComputerSecureChannel -Verbose` |
| Repair secure channel | `Test-ComputerSecureChannel -Repair -Credential $cred -Verbose` |
| Remove from domain (local admin) | `Remove-Computer -WorkgroupName "WORKGROUP" -Force -Restart` |
| Reset computer account password via specific DC | `netdom resetpwd /server:<DC-FQDN> /userd:<domain>\<admin> /passwordd:*` |
