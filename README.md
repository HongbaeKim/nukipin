# NukiPIN

A Node.js integration for managing temporary Nuki smart-lock access codes based on customer bookings.

NukiPIN generates six-digit PINs, assigns validity periods, manages existing access codes, and sends booking instructions by email.

This repository contains the Nuki smart-lock integration from a larger private booking automation system.

## Features

- Generate six-digit access codes based on customer email and booking date.
- Assign PIN validity from 15 minutes before the booking until its scheduled end.
- Reuse the same PIN for multiple bookings by the same customer on the same day.
- Extend access for consecutive bookings.
- Create PINs across configured Nuki smart locks.
- Send access instructions automatically by email.
- Remove expired PINs when processing new bookings.
- Retrieve and inspect existing PINs through an API endpoint.

## Requirements

- Node.js and npm
- Docker and Docker Compose, if using containerized deployment
- Nuki API access token
- Nuki Smart Lock IDs
- SMTP credentials for email delivery

## Installation

### 1. Install Node.js

For Ubuntu or Debian-based systems:

```bash
sudo apt update
sudo apt install -y curl

curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
```

Verify the installation:

```bash
node -v
npm -v
```

Node.js 22 is shown as a modern supported version. Check the project's runtime compatibility before upgrading an existing deployment.

### 2. Install dependencies

Inside the project directory:

```bash
npm install
```

For a new project using the same dependencies:

```bash
npm init -y
npm install express axios nodemailer dotenv
```

### 3. Configure API access

The application requires a Nuki API token and the IDs of the smart locks it manages.

Keep credentials in environment variables or a secure secret store. Do not commit API tokens, SMTP passwords, or customer information to Git.

### 4. Docker deployment

If using Docker Compose, configure the required environment variables and start the services using the repository's Compose configuration.

```bash
docker compose up -d
```

Check container status:

```bash
docker compose ps
```

## Booking scenarios

### 1. Single booking

A customer books a room for one hour.

- A six-digit PIN is generated from the customer's email and booking date.
- Access begins 15 minutes before the booking.
- Access expires at the scheduled end time.
- The PIN is registered on the configured Nuki locks.
- Booking details and access instructions are emailed to the customer.
- Previously expired PINs are cleaned up before creating the new entry.

### 2. Multiple bookings on the same day

Example:

- First booking: 10:00–11:00
- Second booking: 16:00–17:00

The customer receives the same PIN for both bookings because the code is derived from their email and booking date.

Each booking is processed with its own validity period. Depending on how the lock entry is updated, the later booking may replace the earlier access window.

### 3. Consecutive bookings

Example:

- First booking: 13:00–14:00
- Second booking: 14:00–15:00

The same PIN is reused for both bookings.

The updated access period extends through the end of the second booking, allowing the customer to continue using the same code.

### 4. Adjacent bookings by different customers

Example:

| Customer | Booking | PIN validity |
|---|---|---|
| Customer A | 13:00–14:00 | 12:45–14:00 |
| Customer B | 14:00–15:00 | 13:45–15:00 |

Each customer receives a separate PIN.

Because access begins 15 minutes before each booking, both PINs are valid between 13:45 and 14:00.

This overlap is intentional.

### 5. Bookings on different days

A new PIN is generated when the same customer books on another date.

The email and date combination determines the PIN, so bookings on different days produce different codes under the generation scheme.

Expired lock entries are cleaned up when a new booking is processed.

### 6. Multiple rooms

The current implementation does not restrict PINs by room.

The same PIN is registered across the configured locks, allowing access to shared facilities.

The booked room name is included in the confirmation email for reference.

### 7. Additional booking on the same day

If a customer makes another booking on the same day:

- The previously generated PIN is reused.
- The validity period is updated according to the new booking.
- A new confirmation email is sent with the booking details.
- The customer does not need to remember another PIN.

### 8. Expired PIN cleanup

Expired PINs are removed when a new booking is processed.

Cleanup is not currently performed by a separate background scheduler.

### 9. PIN inspection

The following endpoint lists PIN information for configured locks:

```http
GET /list-pins
```

The response provides information such as:

- Lock ID
- PIN
- Associated customer identifier
- Validity start and end times
- Expiration status

This endpoint should only be accessible to authorized administrators because it exposes sensitive access credentials.

## Current limitations

| Feature | Current behavior |
|---|---|
| PIN reuse | Same customer and date reuse the same PIN |
| Consecutive bookings | Existing PIN validity is updated |
| Separate room-level access | Not implemented |
| Separate PIN per lock | Not implemented |
| Scheduled expired-PIN cleanup | Not implemented |
| Cleanup during booking | Implemented |
| Automatic confirmation email | Implemented |

The current design is intended for environments where multiple rooms share common access points.

## Nuki API

### Retrieve Smart Lock IDs

Use the Nuki API to retrieve the locks associated with your account.

```bash
curl -X GET "https://api.nuki.io/smartlock" \
  -H "Authorization: Bearer YOUR_NUKI_API_TOKEN" \
  -H "Content-Type: application/json"
```

Replace `YOUR_NUKI_API_TOKEN` with a valid token.

For regular use, prefer loading the token from an environment variable rather than inserting it directly into shell commands.

```bash
curl -X GET "https://api.nuki.io/smartlock" \
  -H "Authorization: Bearer ${NUKI_API_TOKEN}" \
  -H "Content-Type: application/json"
```

The API response includes the lock identifiers required for configuration.

## Security considerations

NukiPIN manages physical access credentials. Deployments should consider the following:

- Protect API tokens and SMTP credentials.
- Restrict administrative endpoints such as `/list-pins`.
- Never expose PINs in public logs.
- Validate booking data before creating or updating access codes.
- Verify the authenticity of requests from external booking systems.
- Monitor failures when communicating with Nuki devices.
- Ensure expired access credentials are removed according to the intended access policy.

PINs derived from email addresses and dates should not be considered cryptographically unpredictable. A future implementation should use a secure random generator and persistent booking-to-PIN mapping.

## Project scope

NukiPIN is the publicly available smart-lock component of a larger private booking and payment automation system.

This repository focuses on booking-based credential management, Nuki API integration, and automated customer communication. The complete private booking and payment system is not included.
