# MHI2 SSH / Runtime Investigation Environment

This document records the runtime environment used for MHI2 investigation.

> This is an environment reference, not a claim that every command or facility exists on every firmware variant.

## Purpose

SSH/runtime analysis is the bridge between static firmware analysis and MHI2-specific runtime evidence.

Use it to inspect processes and filesystems, observe interfaces, capture logs, inspect DMDT/display state, perform controlled experiments and verify whether static hypotheses correspond to production behaviour.

## Evidence Boundary

Static dump analysis answers:

```text
What exists in the firmware?
```

SSH/runtime analysis answers:

```text
What is actually running and what does it actually do?
```

Do not silently substitute one for the other.

## Known Runtime Areas

### Process / service inspection

Previously used tooling includes QNX process/system inspection such as:

```text
pidin
```

Availability varies by environment. Failed or missing commands should be recorded rather than assuming an equivalent tool exists.

### Logging

The project has used:

```text
sloginfo
sloginfo -w
```

for runtime observation. Record the exact command, filter and context when a log is used as evidence.

### Display / DMDT

The project uses the MHI2 displaymanager debug tooling:

```text
dmdt
```

DMDT evidence must distinguish displayable ID, context ID, physical display/output, routing command and observed result.

Do not infer that a numeric display ID represents a particular physical screen unless runtime observation establishes it.

### Filesystem

Known investigation locations include:

```text
/mnt/app/root
/mnt/system/etc/eso/production
/eso/lib
/eso/bin/apps
```

Exact availability and mount state depend on the runtime image.

### Wireless / Network

Important runtime objects include:

```text
uap0
mlan0
carplay0
ppp0
ecm0
/dev/sdio0
/dev/ipod0
```

Their presence, address, state and ownership should be captured at runtime before using them as evidence.

## Controlled Modification Discipline

Before modifying a production binary or configuration:

1. preserve the original;
2. record its hash;
3. record the exact path;
4. make one controlled change;
5. record the resulting hash;
6. record the installation/restart procedure;
7. capture the runtime result;
8. restore the original unless the experiment explicitly requires otherwise.

Large binary artifacts should not be committed to this repository unless there is a specific reason to do so.

## Runtime Observation Record

For a useful runtime observation, record:

```text
Date/time:
Firmware:
Vehicle/head-unit state:
Command:
Relevant process:
Relevant interface/device:
Expected:
Observed:
Logs/capture:
Evidence ID:
```

## Known Limitations

The runtime environment is constrained. Some QNX commands available in generic documentation may not exist on the target image.

A missing command is not evidence that the underlying facility does not exist.

Likewise, a configuration file found in the filesystem does not prove that the corresponding service is active or that the configuration is currently consumed.

Recovery procedures are intentionally outside this document.
