# F5-RFC5424-Syslog-Normalizer-iRule

An F5 BIG-IP iRule that checks every syslog message on a TCP stream against RFC 5424 and either reports what is wrong or rewrites it into a valid message before it reaches the collector.

**Note that while every effort was taken to process TCP syslog as fast as possible, this solution is CPU intense and only recommended for small environments with limited messages per second requirements.**

Large enterprises should use a dedicated solution for syslog normalization, such as **Cribl.**

## What it checks and fixes

- The presence of no priority value, or a value out of range (0-191). Ensures the value is wrapped in a valid RFC5424 header.
- A priority written with a leading zero which is rewritten without it.
- An old BSD-format message which is converted to RFC 5424 while keeping the hostname, the program name and the process ID of the original message.
- A version field placed in front of an unconverted BSD header, which is what Log Insight Aria sends. 
- A BSD program name and text following an otherwise modern header, the other Aria case. The program name and process ID are split out and the rest becomes the message.
- The rsyslog forwarding format, a BSD layout with an ISO timestamp. This is converted to RFC 5424 formatting.
- A timestamp with a space in place of the `T`, a lowercase `t` or `z`, a fraction longer than six digits, or an offset written without its colon. This is rewritten in canonical form.
- A timestamp with no offset at all. 
- A date that cannot exist, such as February 30th or month 13. This is replaced with the arrival time, keeping the sender's hostname and program name.
- Timestamp and hostname swapped. These are swapped back.
- Structured data sitting in a header field because earlier fields were omitted. The Structured Data is moved back where it belongs for RFC5424.
- A program name still in its BSD form i.e. `sshd[4121]:`.  This is split into the separate program name and process ID fields.
- A header too short to trust. This is wrapped like a message with no header at all.
- A Hostname, program name, process ID or message ID outside printable ASCII or over the length RFC 5424 allows. These bad bytes become underscores and over-long values are simply truncated.
- Malformed structured data. This is replaced with the null value, its original text is kept at the front of the message.
- No structured data field at all; The null value is inserted.
- Messages framed by length count or by line feed, including a sender that mixes the two mid-stream. Detected per connection and resynchronized without losing messages.
- Frames larger than the configured limit. These are passed through unparsed.

Compliant messages are forwarded byte for byte. The only bytes lost are the truncations above, and every change is logged if logging is enabled.

## Settings

In `RULE_INIT` at the top of the rule.

| Variable | Default | Effect |
| --- | --- | --- |
| `syslog_normalizer_repair` | `0` | `0` reports, `1` rewrites non-compliant messages |
| `syslog_normalizer_logging` | `1` | All logging to local0, on or off |
| `syslog_normalizer_log_sample_every` | `100` | First occurrence of each reason combination per connection, then every Nth |
| `syslog_normalizer_output_framing` | `"octet"` | Repair output framing, `octet` or `lf`; must match the collector |
| `syslog_normalizer_default_utc_offset` | `"Z"` | Offset for timestamps carrying none, including every RFC 3164 timestamp |
| `syslog_normalizer_split_rfc3164_tag` | `1` | Split a 3164 tag into APP-NAME and PROCID |
| `syslog_normalizer_max_message_bytes` | `65536` | Largest message parsed; per-connection memory ceiling |


## Logging

Two line types, neither carrying message content:

```
syslog_noncompliant source=10.1.2.3 host=web01 ts=2026-09-19T17:55:40Z reason="legacy BSD RFC 3164 message"
syslog_framing_error source=10.1.2.3 octet-count desync at "...", scanning for next frame
```

## License

MIT License - see [LICENSE](LICENSE) file for details.

## Disclaimer

- This solution is **NOT** officially endorsed, supported, or maintained by F5 Inc.
- F5 Inc. retains all rights to their trademarks, including but not limited to "F5", "BIG-IP", "TMOS", "SSL Orchestrator", and related marks
- This is an independent, community-developed solution that utilizes F5 products but is not affiliated with F5 Inc.
- For official F5 support and solutions, please contact F5 Inc. directly

**Technical Disclaimer:**
- This software is provided "AS IS" without warranty of any kind
- The authors and contributors are not responsible for any damages or issues that may arise from its use
- Always test thoroughly in non-production environments before deployment
- Backup your F5 configuration before implementing any changes
- Review and understand all code before deploying to production systems

By using this software, you acknowledge that you have read and understood these disclaimers and agree to use this solution at your own risk.
