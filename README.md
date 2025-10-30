# IBM MQ CHLAUTH & MCAUSER Implementation and Troubleshooting Guide

---

© 2025 Shivaraj — All Rights Reserved. 

## 🧩 1. Concept Overview

### 1.1 What is CHLAUTH (Channel Authentication)?
**Channel Authentication Records (CHLAUTH)** are IBM MQ security features that control who can connect to your queue manager via channels. They help protect MQ from unauthorized access.

**In simple terms:**
- CHLAUTH acts like a firewall for MQ channels.
- It allows, blocks, or maps connections based on user ID, IP address, SSL, or client type.

**Technically:**
- Rules are stored in the queue manager object `SYSTEM.CHLAUTH.DATA.QUEUE`.
- Evaluated when a client tries to connect using a channel (e.g., `SVRCONN`).

### 1.2 What is MCAUSER?
**MCAUSER (Message Channel Agent User)** defines which user ID the channel uses to access MQ resources **after** authentication.

**Non-technical analogy:**
> Think of MCAUSER as the badge that the channel wears after entering MQ. Even if User1 connects, MQ will treat it as the MCAUSER if set.

**Technically:**
- If `MCAUSER` is blank, MQ uses the ID provided by the client connection.
- If `MCAUSER` is set, all operations on that channel are executed as that ID.

---

## ⚙️ 2. Checking and Enabling CHLAUTH

### 2.1 Check if CHLAUTH is enabled
```bash
DISPLAY QMGR CHLAUTH
```
**➡️ Shows whether channel authentication rules are active (ENABLED or DISABLED).**

### 2.2 Enable CHLAUTH
```bash
ALTER QMGR CHLAUTH(ENABLED)
```
**➡️ Turns on channel-level security.**

### 2.3 Disable CHLAUTH (not recommended for production)
```bash
ALTER QMGR CHLAUTH(DISABLED)
```
**➡️ Turns off all channel authentication checks.**

### 2.4 List all CHLAUTH rules
```bash
DISPLAY CHLAUTH(*) ALL
```
**➡️ Displays all existing channel authentication records and details.**

---

## 🧠 3. Defining CHLAUTH Rules

### 3.1 Block a specific user
```bash
SET CHLAUTH('APP.TO.MQ') TYPE(BLOCKUSER) USERLIST('baduser')
```
**➡️ Prevents user `baduser` from connecting via channel `APP.TO.MQ`.**

### 3.2 Block all users (except those mapped)
```bash
SET CHLAUTH('APP.TO.MQ') TYPE(BLOCKUSER) USERLIST('nobody')
```
**➡️ Default setting to block no one — 'NOBODY' means no blocking.**

### 3.3 Map connections from specific IPs to a user
```bash
SET CHLAUTH('APP.TO.MQ') TYPE(ADDRESSMAP) ADDRESS('10.10.10.*') USERSRC(MAP) MCAUSER('appusr')
```
**➡️ All clients from 10.10.10.x network connect as user `appusr`.**

### 3.4 Allow only a specific user
```bash
SET CHLAUTH('APP.TO.MQ') TYPE(USERMAP) CLNTUSER('mqapp') USERSRC(MAP) MCAUSER('mqapp')
```
**➡️ Only user `mqapp` can connect to this channel.**

### 3.5 Delete a CHLAUTH rule
```bash
SET CHLAUTH('APP.TO.MQ') TYPE(USERMAP) CLNTUSER('mqapp') ACTION(REMOVE)
```
**➡️ Removes a specific rule for the channel.**

---

## 👥 4. MCAUSER Deep Dive

| Scenario | MCAUSER Setting | Effective User | Explanation |
|-----------|------------------|----------------|--------------|
| `MCAUSER(mqm)` | Explicit | mqm | All connections act as mqm (admin). Dangerous if exposed. |
| `MCAUSER(appusr)` | Explicit | appusr | All connections run with `appusr` permissions. |
| `MCAUSER('')` (blank) | None | Client user | MQ uses the user ID from the client connection. |

### 4.1 Display current MCAUSER for a channel
```bash
DISPLAY CHL(APP.TO.MQ) MCAUSER
```
**➡️ Shows the effective MCAUSER assigned to the channel.**

### 4.2 Change MCAUSER for a channel
```bash
ALTER CHL(APP.TO.MQ) CHLTYPE(SVRCONN) MCAUSER('appusr')
```
**➡️ Forces all channel connections to act as `appusr`.**

---

## 🔍 5. Validation Commands

### 5.1 Test which rule applies to a connection
```bash
DISPLAY CHLAUTH MATCH(RULE) ADDRESS('10.10.10.11') CLNTUSER('user1') CHANNEL('APP.TO.MQ')
```
**➡️ Shows which CHLAUTH rule would apply to a given connection attempt.**

### 5.2 Display effective authority of a user
```bash
dspmqaut -m QM1 -t q -n TEST.Q -p appusr
```
**➡️ Displays permissions of `appusr` on queue `TEST.Q`.**

### 5.3 List all users with permissions
```bash
dspmqaut -m QM1 -t q -n TEST.Q
```
**➡️ Shows all users with access rights to queue `TEST.Q`.**

---

## 🧯 6. Troubleshooting CHLAUTH & Connection Issues

| Error | Meaning | Fix |
|--------|----------|------|
| `MQRC_NOT_AUTHORIZED (2035)` | User lacks authority or blocked by CHLAUTH | Check MCAUSER, CHLAUTH rule, and setmqaut permissions |
| `MQRC_UNKNOWN_ENTITY (2292)` | User or group not recognized | Ensure user exists on MQ host system |
| `MQRC_SECURITY_ERROR (2063)` | SSL or CHLAUTH mismatch | Verify SSLPEER, CERTLABL, and CHLAUTH filters |
| Channel not starting | Mapped to invalid user | Check if MCAUSER exists and has MQ permissions |

### 6.1 Disable CHLAUTH temporarily (for troubleshooting only)
```bash
ALTER QMGR CHLAUTH(DISABLED)
REFRESH SECURITY TYPE(CONNAUTH)
```
**➡️ Temporarily disables CHLAUTH rules for debugging. Re-enable after.**

### 6.2 Refresh channel authentication cache
```bash
REFRESH SECURITY TYPE(CHLAUTH)
```
**➡️ Applies recent CHLAUTH changes immediately.**

---

## 🛡️ 7. Best Practices for Production

✅ Always keep `CHLAUTH(ENABLED)` in production.  
✅ Avoid `MCAUSER(mqm)` on SVRCONN channels.  
✅ Use service accounts (e.g., `mqapp`, `mqread`) for each application.  
✅ Set `SSLCAUTH(REQUIRED)` if SSL is used.  
✅ Audit CHLAUTH and authority mappings regularly:  
```bash
DISPLAY CHLAUTH(*) ALL
```
✅ Refresh CHLAUTH cache after modifications:  
```bash
REFRESH SECURITY TYPE(CHLAUTH)
```

---

## 🧰 8. Quick Command Reference

| Purpose | Command |
|----------|----------|
| Check CHLAUTH status | `DISPLAY QMGR CHLAUTH` |
| Enable CHLAUTH | `ALTER QMGR CHLAUTH(ENABLED)` |
| Display all rules | `DISPLAY CHLAUTH(*) ALL` |
| Create rule (USERMAP) | `SET CHLAUTH('CHL') TYPE(USERMAP) CLNTUSER('x') USERSRC(MAP) MCAUSER('y')` |
| Block user | `SET CHLAUTH('CHL') TYPE(BLOCKUSER) USERLIST('baduser')` |
| Refresh security | `REFRESH SECURITY TYPE(CHLAUTH)` |
| Display MCAUSER | `DISPLAY CHL('CHL') MCAUSER` |
| Change MCAUSER | `ALTER CHL('CHL') CHLTYPE(SVRCONN) MCAUSER('appusr')` |
| Check user permissions | `dspmqaut -m QM1 -t q -n Q1 -p appusr` |

---

## 📘 References
- [IBM MQ Official Documentation: Channel Authentication Records](https://www.ibm.com/docs/en/ibm-mq/latest?topic=features-channel-authentication-records)
- [IBM MQ Security Concepts](https://www.ibm.com/docs/en/ibm-mq/latest?topic=security-introduction)
- [IBM MQ Command Reference](https://www.ibm.com/docs/en/ibm-mq/latest?topic=reference-mqsc-commands)

---

✅ **Prepared for:** IBM MQ Admins and Security Engineers  
✅ **Validated on:** IBM MQ 9.2.x / 9.3.x / 9.4.x  
✅ **Environment:** RHEL 8.x / 9.x, MQ on AWS RDQM / standalone

## 📞 Contact
- shivaraj
- shivantu9@gmail.com
- For questions or contributions, please reach out via GitHub issues.

---

