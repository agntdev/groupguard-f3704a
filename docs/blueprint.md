# GroupGuard — Bot specification

**Archetype:** community

**Voice:** professional and concise — write every user-facing message, button label, error, and empty state in this voice.

A Telegram group moderation bot that automates anti-spam enforcement, new member verification, and provides admin tools for managing group rules and tracking moderation actions. Sends alerts and reports to admins while maintaining a simple, user-friendly interface for members.

> This is the complete contract for the bot. Implement EVERY entry point, flow, feature, integration, and edge case below. The completeness review checks the bot against this document after each build pass.

## Primary audience

- Telegram group admins
- members of public/private groups

## Success criteria

- Automated verification of new members with configurable timeout
- Effective anti-spam enforcement with configurable thresholds
- Admins can manage rules and view logs/reports
- Clear explanations for automated actions in-group
- Regular audit logs and summary reports to admins

## Entry points

Every feature must be reachable from the bot's command/button surface (button-first; only /start and /help are slash commands).

- **/start** (command, actor: user, command: /start) — Open the main menu
- **I'm human** (button, actor: user, callback: verify:human) — Verify new members by confirming they're human
  - inputs: member_id, join_time
  - outputs: verification_status, message_posted
- **/warn** (command, actor: admin, command: /warn) — Issue a warning to a user
  - inputs: target_user, reason
  - outputs: infraction_record, message_posted
- **/mute** (command, actor: admin, command: /mute) — Mute a user for a specified duration
  - inputs: target_user, duration
  - outputs: infraction_record, message_posted
- **/kick** (command, actor: admin, command: /kick) — Kick a user from the group
  - inputs: target_user, reason
  - outputs: infraction_record, message_posted
- **/ban** (command, actor: admin, command: /ban) — Ban a user from the group
  - inputs: target_user, reason
  - outputs: infraction_record, message_posted
- **/trust** (command, actor: admin, command: /trust) — Mark a user as trusted (exempt from automated actions)
  - inputs: target_user
  - outputs: trusted_status, message_posted
- **/untrust** (command, actor: admin, command: /untrust) — Remove a user's trusted status
  - inputs: target_user
  - outputs: trusted_status, message_posted
- **/setwelcome** (command, actor: admin, command: /setwelcome) — Set the welcome message for new members
  - inputs: message_text
  - outputs: welcome_message
- **/setrules** (command, actor: admin, command: /setrules) — Set the rules message for the group
  - inputs: message_text
  - outputs: rules_message
- **/setautoactions** (command, actor: admin, command: /setautoactions) — Configure automated action thresholds
  - inputs: action_type, threshold
  - outputs: autoaction_config
- **/log** (command, actor: admin, command: /log) — View recent moderation actions
  - inputs: limit
  - outputs: action_log
- **/report** (command, actor: admin, command: /report) — View summary statistics and reports
  - inputs: report_type
  - outputs: summary_report

## Flows

### New member verification
_Trigger:_ new_member_joined

1. Send welcome message with 'I'm human' button
2. Wait for verification within timeout window
3. If verified, lift restrictions and post confirmation
4. If not verified, remove user and log removal

_Data touched:_ Member, Verification challenge

### Anti-spam monitoring
_Trigger:_ message_posted

1. Check if message contains suspicious links from new accounts
2. Check for duplicate messages within time window
3. Check for message flood patterns
4. If thresholds exceeded, escalate actions (warn → mute → kick/ban)
5. Post explanation for action in-group

_Data touched:_ Infraction, Member

### Admin command execution
_Trigger:_ /warn, /mute, /kick, /ban, /trust, /untrust, /setwelcome, /setrules, /setautoactions, /log, /report

1. Validate admin permissions
2. Execute command
3. Update relevant data entities
4. Post confirmation message

_Data touched:_ Member, Infraction, Action log

## Data entities

Durable data (must survive a restart) uses the toolkit's persistent store, never in-memory maps.

- **Member** _(retention: persistent)_ — User who has joined the group
  - fields: user_id, join_timestamp, verification_state, trusted_flag, infraction_count
- **Verification challenge** _(retention: session)_ — Verification process for new members
  - fields: challenge_id, member_id, timeout_window, verification_status
- **Infraction** _(retention: persistent)_ — Record of detected issues or violations
  - fields: infraction_id, member_id, timestamp, violation_type, severity, action_taken
- **Action log** _(retention: persistent)_ — Log of moderator or automated actions
  - fields: log_id, actor, target, action_type, reason, timestamp, source

## Integrations

- **Telegram** (required) — Bot API messaging and group management
Call external APIs against their real contract (correct endpoints, ids, params); credentials from env. Do not fake responses.

## Owner controls

- Configure verification timeout
- Set protected account age for link checks
- Adjust duplicate message thresholds
- Set flood detection thresholds
- Configure auto-action escalation rules
- Set welcome message
- Set rules message
- Choose report destination (DM or channel)

## Notifications

- Welcome message with verification prompt
- Verification timeout warning
- Automated action explanations in-group
- Alerts for removals and high-severity events
- Periodic summary reports to admin

## Permissions & privacy

- Bot has access to group messages and member list
- Only admin can configure bot settings
- Automated actions are applied to non-admin members only
- Verification status and infraction data is stored securely
- Action logs are retained for audit purposes

## Edge cases

- Multiple new members joining simultaneously
- Admins attempting to trigger automated actions on themselves
- Users attempting to bypass verification with multiple accounts
- Rapid succession of automated actions requiring rate limiting
- Message floods that exceed multiple thresholds simultaneously

## Required tests

- End-to-end verification flow with timeout handling
- Anti-spam detection with various violation types
- Admin command execution and permissions validation
- Automated action escalation sequence
- Report generation and formatting

## Assumptions

- Verification timeout is 5 minutes by default
- Protected account age for link checks is 7 days by default
- Duplicate message threshold is 2 messages in 30 seconds
- Flood threshold is 5 messages in 10 seconds
- Default auto-action sequence is warn → mute → kick
- Admins are exempt from automated actions
- Automated actions are explained in-group
- Report destination is owner's DM by default
