# Authentication & Account Lifecycle

## Registration

1. Class Leader or Commissioner uploads college data (registration numbers + basic info).
2. Student enters their registration number on the public registration form.
3. If the number exists and is unused, they complete their profile (email, phone, password, state).
4. Account is created with `is_verified = False`.
5. Commissioner must verify the account before the student can vote.

## Login

- Uses registration number as the username field.
- Strong brute-force protection (Redis).
- After successful login, user is redirected to their role-appropriate dashboard.

## Password Recovery

Three supported flows (all state managed in Redis with short TTLs):

- Classic email + token link
- Security questions (randomly sampled from a configurable set)
- Phone-based OTP (if SMS integration is enabled)

## Verification

Only verified users with active `VoterToken`s can cast votes.

Commissioners can verify users individually or in bulk.

## Tokens for Voting

Upon verification (or election activation), eligible `VoterToken` records are created for each election level the user is allowed to participate in.

These tokens are single-use and consumed atomically when votes are submitted.
