# Important User & System Flows

## 1. Registration Flow
1. Class leader uploads college data (Excel/CSV)
2. Student visits registration page
3. Enters registration number → validated against unused `CollegeData`
4. Completes profile (email, phone, password, state)
5. Account created in `is_verified=False` state
6. Commissioner verifies the account

## 2. Voting Flow
See `diagrams/secure-voting.mmd` for the detailed sequence.

## 3. Results Publication
1. Election period ends (or is closed)
2. Commissioner reviews (internal tallies available to privileged roles)
3. Commissioner sets `results_published = True`
4. All users can now view full results and download PDF report

## 4. Password Recovery
Multiple paths supported:
- Classic email token
- Security questions
- Phone OTP (SMS)

All state managed in Redis with short TTLs + rate limiting.

## 5. Bulk Data Upload
- Class leader / commissioner uploads file
- Creates `CollegeDataUploadBatch`
- Async processing (or sync in current flows)
- Generates user accounts + initial passwords via email

Refer to the main diagrams for visual representations.
