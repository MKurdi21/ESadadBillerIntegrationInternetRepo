# E‑Sadad Biller Integration

This repository contains a middleware service that links a biller system with the **E‑Sadad** payment platform. The solution follows clean architecture principles with separate projects for the domain (`Esadad.Core`), infrastructure (`EsadadInfrastructure`) and the ASP.NET Core API (`EsadadAPI`).

## Features
- Prepaid validation, post‑paid bill pull and payment notification APIs with XML and JSON variants.
- Communicates with the internal billing system either through REST APIs or by reading and writing directly to its database.
- Uses query string parameters and conventional URL routing; API versioning is recommended for future expansion.
- Digital signature verification and certificate management.
- Configuration files for biller metadata, service definitions and certificate information.
- SQL Server logging for requests, responses and payment details.
- Handles decimal precision per currency (JOD: 3 decimals, USD: 2, ILS: 2).

## Configuration
The service reads its configuration from `appsettings.json`:

- **BillerInfo** – `BillerCode`, `BillerName`.
- **Services** – per‑service settings such as `ServiceTypeCode`, Arabic and English names, payment type (PostPaid or PrePaid), currency, `BankCode`, `IBAN`, whether partial payment is allowed, and optional `LowerValue`/`UpperValue` limits.
- **Certificates** – owner (E‑Sadad or Biller), type (Public or Private), password and file path.

## Logging Tables
Entity Framework Core persists activity in SQL Server tables:

- `EsadadTransactionsLogs` records each request and response with fields such as transaction type, API name, GUID, billing number, bill number, service type, currency and payload.
- `EsadadPaymentsLogs` stores payment information including GUID, billing number, bill number, service type, currency, amount and whether the payment was applied.

## API Workflows

### Bill Pull
1. A request provides a `guid` query parameter and an XML body; optional `username` and `password` may also be included.
2. The request is logged in `EsadadTransactionsLogs`.
3. Inputs are validated and the request signature is checked.
4. If the signature is invalid, an error XML response is returned.
5. With a valid signature, the middleware queries the billing system:
   - If the billing number is unknown, an invalid billing number response is returned.
   - If a due amount exists, a bill with the amount is generated.
   - If no amount is due, a bill with zero due is returned.
   - Any unexpected error generates a generic error response.
6. The response body is signed and saved to `EsadadTransactionsLogs`.
7. The signed XML response is returned.

### Receive Payment Notification
1. The request supplies a `guid` query parameter and an XML body; `username` and `password` are optional.
2. The request is logged in `EsadadTransactionsLogs` and a record is created in `EsadadPaymentsLogs` with `IsPaid` set to false to support retry handling.
3. Inputs are validated and the signature is verified.
4. If the signature is invalid, an error XML response is returned.
5. With a valid signature, the middleware sends the payment to the billing system:
   - If the payment GUID has not been processed, the payment is executed, the `IsPaid` flag is set to true and a success response is generated.
   - If the payment was already applied, a success response is returned without reprocessing.
6. The response body is signed and written to `EsadadTransactionsLogs`.
7. The signed XML response is returned.

## Project Structure
The solution `EsadadBillerIntegrationAPi.sln` includes three projects:

| Project | Purpose |
|---------|---------|
| `Esadad.Core` | Domain entities and models used across the service. |
| `EsadadInfrastructure` | DTOs, helpers, EF Core context and service implementations. |
| `EsadadAPI` | ASP.NET Core API exposing prepaid, postpaid, payment and authentication endpoints. |

## Getting Started
1. **Install prerequisites**
   - .NET 9 SDK (preview) and SQL Server.
   - Place required certificates under `EsadadAPI/Certs` and update paths and passwords in `EsadadAPI/appsettings.json`.
2. **Restore and build**
   ```bash
   dotnet restore EsadadBillerIntegrationAPi.sln
   dotnet build EsadadBillerIntegrationAPi.sln
   ```
3. **Run the API**
   ```bash
   dotnet run --project EsadadAPI/Esadad.API.csproj
   ```

For detailed instructions, see [USAGE_GUIDE.md](USAGE_GUIDE.md).

## Security Notes
Connection strings, JWT secrets and certificate passwords are stored in `appsettings.json` for development only. Use environment variables or a secret manager for production deployments.

