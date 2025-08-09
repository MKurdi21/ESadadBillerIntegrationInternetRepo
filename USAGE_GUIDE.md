# ESadad Biller Integration – Usage Guide

This document outlines the steps to configure, build and run the ESadad biller integration service.

## 1. Prerequisites
- **.NET SDK**: The solution targets .NET 9.0. Install the matching SDK from [Microsoft](https://aka.ms/dotnet-download). The container used for this report includes .NET 8.0 only, so building fails (see Testing section for details).
- **SQL Server**: Used for transaction logging. Update the connection string in `EsadadAPI/appsettings.json` to point to your instance.
- **Certificates**: `ESADADTEST.cer` and `BillerCertPfx.pfx` must exist under `EsadadAPI/Certs` with the passwords configured in `appsettings.json`.

## 2. Restore and Build
```bash
# From the repository root
dotnet restore EsadadBillerIntegrationAPi.sln
# Build (requires .NET 9 SDK)
dotnet build EsadadBillerIntegrationAPi.sln
```

## 3. Running the API
```bash
# Run the ASP.NET Core API
dotnet run --project EsadadAPI/Esadad.API.csproj
```
The API exposes Swagger UI in development environments for interactive exploration.

## 4. Configuration Overview
Runtime settings reside in `EsadadAPI/appsettings.json`:
- SQL Server connection string
- Paths and passwords for certificates
- JWT authentication settings
- Biller service metadata and currency rules

Use environment variables or secret stores for production deployments to avoid committing credentials to source control.

## 5. API Endpoints
| Endpoint | Description |
|----------|-------------|
| `POST /prepaid/PrepaidValidation?GUID={guid}` | Validates prepaid accounts using an XML payload and digital signatures. |
| `POST /postpaid/BillPull?GUID={guid}` | Retrieves billing details for postpaid accounts. |
| `POST /payment/ReceivePaymentNotification?GUID={guid}` | Records payment notifications. |
| Public JSON equivalents | Available under `/public/*` for JSON-based requests. |
| `POST /Authentication` | Validates credentials and returns a JWT token for subsequent calls. |

## 6. Logging and Persistence
Requests, responses and payment records are stored via Entity Framework Core in the following tables:
- `EsadadTransactionLog` and `EsadadTransactionJsonLog`
- `EsadadPaymentLog`

## 7. Troubleshooting
- Ensure the installed .NET SDK supports the targeted framework. The build will fail with **NETSDK1045** when attempting to target .NET 9 with the .NET 8 SDK.
- Confirm the certificates match the configuration; signature validation fails otherwise.
- Check SQL connectivity if logging does not occur.
