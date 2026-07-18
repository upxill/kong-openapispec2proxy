# Kong OpenAPI Proxy with decK

This project shows how to convert an OpenAPI Specification into a Kong declarative proxy configuration using decK and run it locally.

## Files

- [open-api-spec.yaml](open-api-spec.yaml) — source OpenAPI 3.1 definition
- [kong.yaml](kong.yaml) — generated Kong declarative config

## Prerequisites

Make sure you have the following installed:

- Docker Desktop
- decK CLI
- Kong container image

## Design Steps

The workflow for this proxy can be broken into simple design stages:

1. Define the API contract in the OpenAPI spec.
2. Map OpenAPI paths and methods to Kong routes and services.
3. Point the Kong service to the upstream backend host.
4. Generate a declarative Kong configuration with decK.
5. Validate, sync, and run the configuration in a local Kong instance.
6. Add security and traffic-control plugins as needed for production use.

## 1. Convert the OpenAPI spec to Kong config

From the project folder, run:

```bash
deck file openapi2kong open-api-spec.yaml > kong.yaml
```

This generates a Kong declarative configuration containing:

- a service for the upstream API
- routes for each OpenAPI path
- an upstream target for the backend host

## 2. Review the generated config

The generated file is saved as [kong.yaml](kong.yaml). It includes route definitions such as:

- `/eligibility/check`
- `/claims/submit`

## 3. Start Kong locally

Run Kong in DB-less mode with the generated config:

```bash
docker run -d --name kong-openapi-proxy \
  -e "KONG_DATABASE=off" \
  -e "KONG_DECLARATIVE_CONFIG=/usr/local/kong/kong.yaml" \
  -e "KONG_PROXY_LISTEN=0.0.0.0:8000" \
  -e "KONG_ADMIN_LISTEN=0.0.0.0:8001" \
  -v "$PWD/kong.yaml:/usr/local/kong/kong.yaml" \
  -p 8000:8000 \
  -p 8001:8001 \
  kong:latest
```

## 4. Validate the config

```bash
deck gateway validate kong.yaml
```

## 5. Sync the config to Kong

```bash
deck gateway sync kong.yaml --kong-addr http://127.0.0.1:8001
```

## 6. Test the proxy

Example request for eligibility checks:

```bash
curl -X POST http://127.0.0.1:8000/eligibility/check \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "providerNpi": "1987654321",
    "memberId": "MED123456789",
    "dateOfService": "2026-07-18"
  }'
```

Example request for claims submission:

```bash
curl -X POST http://127.0.0.1:8000/claims/submit \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "claimId": "a80b8751-2461-41d9-813c-cc436df7a0cc",
    "providerNpi": "1987654321",
    "memberId": "MED123456789",
    "diagnosisCodes": ["M54.50", "R51.9"],
    "lineItems": [
      {
        "serviceDate": "2026-07-18",
        "procedureCode": "99213",
        "chargedAmount": 125.00
      }
    ]
  }'
```

## Notes

- This example uses the declarative DB-less Kong approach.
- For production, you can add authentication, rate limiting, and plugin policies to the generated config.
- If you want to make the proxy more secure, add Kong plugins such as JWT validation, key-auth, or rate-limiting.
