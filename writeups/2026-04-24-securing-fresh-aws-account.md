# Securing a Fresh AWS Account: Lessons from an MFA Lockout Recovery

**Date:** April 24, 2026
**Topics covered:** Root user, IAM users, MFA, authenticator apps, account recovery, least privilege

## What this writeup is

This is an honest walkthrough of setting up my AWS account properly — including the part where I got locked out and had to recover access. 
Instead of a sterile "how to" guide, I wanted to document what actually happened, including the mistakes, because that's how real cloud work goes.

## Why I did this

I set up my AWS account around 9 months ago while studying for the Solutions Architect Associate exam, but I hadn't touched it since. 
When I logged back in to start using it seriously, I realized two things:

1. I'd been using the root user for everything, which is a no-no in the industry.
2. The MFA method I'd set up — a passkey — wasn't accessible on any device I currently owned.

Both needed fixing before I did any real work in AWS.

## The MFA recovery problem

When I tried to log in, AWS asked me to authenticate with a passkey I didn't recognize. I tried scanning the QR code with my phone, 
but my phone reported "no passkey found." I'd apparently set up a passkey 9 months ago that either never synced to my phone properly or 
was wiped when I changed devices or browser data.

**What I learned:** Passkeys are convenient, but they can quietly disappear if the device they're stored on gets reset, replaced, or signs out of its syncing account. 
For something as critical as AWS root access, I switched to an authenticator app (Google Authenticator) — not because it's *more secure*, but because the recovery story is more transparent.

Worth being precise about the trade-off: passkeys are theoretically more secure than TOTP — they're phishing-resistant and can't be cloned. 
The reason I went with TOTP isn't security strength; it's recovery transparency. 
I save the secret key, I can restore on any new device, no cloud-sync magic required. 
For higher-stakes accounts — production environments, financial, work AWS — a FIDO2 hardware key like a YubiKey is the gold standard: phishing-resistant like a passkey, but bound to physical hardware I control.

Let's just say I'll be putting the secret key in a safe place from now on!

### How I recovered access

AWS has an "alternative factors" recovery flow buried behind a "Trouble signing in?" link on the MFA prompt. It required:

1. Email verification (code sent to my account email)
2. Phone verification (SMS code sent to the phone number on the account)

Both of those had to be reachable and working. If either had been outdated, I would have been looking at a full support ticket that could take days.

**Lesson:** Always keep the recovery email and phone number on any cloud account current. These are the lifelines when your primary auth fails.

## Hardening the account after recovery

Once I was back in, I did the following in order:

### 1. Removed the broken passkey

Deleted the inaccessible passkey from the root user's security credentials. Keeping dead MFA devices around is bad hygiene — 
they clutter the security review of an account and can mask real issues during an audit. Anyone reviewing the account should see only 
active, intentional credentials.

### 2. Set up Google Authenticator for the root user

Added a virtual MFA device using Google Authenticator. This time, I saved the secret key — the text string alongside the QR code — in a password manager. 
That secret is functionally a password (anyone with it can generate valid codes), so the right home is an encrypted vault, not a plain text file or a sticky note. 
If I lose my phone, I can restore the MFA on a new device without another recovery ordeal.

### 3. Created an IAM user for day-to-day work

Per AWS best practice, the root user should only be used for a handful of account-level tasks (billing changes, account closure, etc.). 
Everything else should go through IAM users with scoped permissions.

Steps taken:
- Created a user group called `Admins` with the `AdministratorAccess` policy attached
- Created an IAM user and added it to the `Admins` group
- Enabled MFA on the IAM user separately (MFA does not transfer between users)
- Saved the IAM sign-in URL (different from root's sign-in URL) for future logins

It's reassuring to have a daily-driver account that lacks root's account-level powers.

### 4. Enabled IAM access to billing information

By default, IAM users — even ones with AdministratorAccess — cannot see billing data. I found this out when the AWS console home page 
greeted me with three "Access denied" errors on the cost widgets. Fixing this requires a toggle in the root account's billing preferences:

> Account → IAM user and role access to Billing information → Activate IAM Access

This is a good example of AWS's separation-of-concerns design: some account-level functions are deliberately gated to the root user 
regardless of IAM permissions. Billing access is one of them — the toggle has to be flipped by root explicitly before any IAM user 
can see it, even an admin.

## Takeaways

- **Passkeys are convenient but fragile when tied to a single device.** For critical accounts, an authenticator app with a saved backup key
  is more resilient.
- **Test your recovery paths before you need them.** If I'd tried logging in even once in the last 9 months, I would have caught the passkey
  issue before it was urgent.
- **Root should be used rarely.** Creating an IAM admin user and logging in as that user going forward is both best practice and a habit
  worth building now.
- **AdministratorAccess on a personal account is a deliberate simplification.** In a real production environment, you'd scope permissions
  much more tightly. I'm flagging that trade-off here so future me doesn't forget.

## Why this matters for security engineering

Identity hygiene — root usage, MFA enforcement, IAM least privilege, recovery path testing — is the first thing a Cloud Security Engineer 
audits in any AWS account.
Getting these wrong is the most common cause of "account compromised" tickets in cloud organizations.
The recovery ordeal I went through was a mild version of what a Security Engineer walks a non-technical user through during an actual incident.

The lessons here translate directly to the roles I'm targeting:

- **Root user activity is a flag in any AWS audit.** CloudTrail logs that show root API calls are typically escalated to security teams 
  immediately, because there's almost no legitimate reason for root to be making day-to-day changes.
- **MFA on every privileged identity is non-negotiable** in compliance frameworks like SOC 2 and ISO 27001.
- **Recovery paths are part of the security posture.** A strong auth setup that locks legitimate users out is just downtime in fancier dress.

A note on how MFA audit findings have evolved: AWS now enforces MFA on root users across all account types (rolled out from May 2024 through June 2025), so "no MFA on root" has largely disappeared as an audit finding on accounts created since. 
What's replaced it is the failure mode I just described: MFA enabled, but stored on a device the user can no longer access, with stale recovery factors. 
The auth method passes the audit checkbox; the recovery posture doesn't.

## What's next

Next I'll set up a billing alarm and then move on to hands-on work with EC2. Both will get their own writeups.
