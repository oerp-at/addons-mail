# mail_composer_cc_bcc

Separate To, Cc and Bcc recipients in the mail composer (`mail.compose.message`).

## What it adds

- `res.company`: `default_partner_cc_ids` / `default_partner_bcc_ids`, prefilled in the
  composer.
- `mail.template`: `email_bcc` (placeholders allowed). Template `email_cc` / `email_bcc`
  are resolved to partners and put into the composer Cc/Bcc fields.
- Composer: `partner_cc_ids`, `partner_bcc_ids`.
- `mail.message`: `recipient_cc_ids`, `recipient_bcc_ids` (who got the mail as Cc/Bcc).
- `mail.mail`: `email_bcc`.

## How sending works

- Only mails sent from the composer are affected: `_action_send_mail_comment` sets the
  context key `is_from_composer` plus the Cc/Bcc partners; everything else (chatter
  notifications, mass mailing, automatic mails) keeps the core behaviour.
- One `mail.mail` per recipient, but every mail carries the same To and Cc headers, so
  mail clients show the expected recipient list. Bcc recipients get an `X-Odoo-Bcc`
  header instead of `Bcc` (Python's `smtp.send_message` drops `Bcc`).
- `ir.mail_server._prepare_email_message` replaces the SMTP envelope (`RCPT TO`) with
  the next address from the context key `recipients`, one per outgoing mail.
- `skip_adding_cc_bcc` in the context disables the Cc/Bcc handling; `_notify_thread`
  sets it for messages of type `notification`.

## Pitfalls

- Logic depends on the order of mails produced by `mail.mail._prepare_outgoing_list` and
  on the `recipients` list popped in `ir.mail_server`; core changes in these methods
  break it silently. Check them first on every migration.
- `_compute_partner_ids` is a copy of the core method without `email_cc`.
  `test_MailComposer_upstream_file_hash` fails when core changes it: port the core diff
  into the copy, then add the new md5 to `VALID_HASHES` (keep the old ones for older
  Odoo/OCB builds).
- Odoo 20.0 core has its own Cc (`partner_cc_ids` on the composer, `partner_cc` on
  templates, `recipient_cc_ids` on `mail.mail`). Field names overlap with this module,
  so a 20.0 port must build on core Cc and only add Bcc and company defaults.
