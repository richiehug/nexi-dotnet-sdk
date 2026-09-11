# OPI .NET SDK

A .NET SDK for integrating OPI-compatible payment terminals into Windows POS and ECR applications.

## Installation

Download Full or Lite from the [latest release](https://github.com/richiehug/opi-dotnet-sdk/releases/latest) and [add the local DLL](integration.md#add-the-sdk-to-an-application). Choose one edition; do not combine their DLLs. Lite includes its required Microsoft dependency DLLs.

## Requirements

- Windows
- Full: .NET 10
- Lite: .NET Framework 4.8 or 4.8.1

## Quick start

```csharp
using Opi.Sdk;

await using var opiSdk = new OpiClient(new OpiOptions
{
    TerminalHost = "192.168.0.69",
    WorkstationId = "CASHIER-01"
});

async Task TakePaymentAsync()
{
    var result = await opiSdk.PaymentAsync(
        amount: 1250,
        currency: "CHF"
    );

    switch (result.Status)
    {
        case ResultState.Success:
            ShowSuccess(result);
            break;
        case ResultState.Declined:
            ShowDecline(result.ErrorCode);
            break;
        default:
            ShowPaymentError(result);
            break;
    }
}
```

Amounts use the currency's minor unit, so `1250` represents CHF 12.50. The `Show…` methods above are supplied by your cashier application.

## What the SDK provides

- Take payments, issue refunds, and reverse transactions through a clean C# API.
- Run terminal operations and retrieve terminal information directly from your app.
- Handle every outcome with typed results and structured errors.
- Print merchant and customer receipts on the terminal or return them to your app for display and storage.
- Receive Dynamic Currency Conversion (DCC) details when offered and returned by the terminal.
- Use the payment experience in English, German, French, or Italian.
- Configure timeouts, receipt handling, and Force Acceptance to match your integration.
- Return uncertain transaction results for ECR-controlled reconciliation.
- Diagnose integration issues with optional Normal summaries or Extended sanitized protocol diagnostics; logging defaults to off.

## Documentation

- [Integration guide and API overview](integration.md)
- [Detailed C# API reference](https://richiehug.github.io/opi-dotnet-sdk/)

The integration guide covers installation, configuration, all public terminal methods, request and response models, result states, errors, receipts, recovery, and operation events. XML IntelliSense documentation is included with the DLLs.

## Requirements

- Full: a Windows .NET 10 application
- Lite: a Windows .NET Framework 4.8 or 4.8.1 application
- Asynchronous C# support; see the integration guide for older C# syntax
- A provisioned OPI-compatible payment application reachable through the terminal's network address

## Support

Use [GitHub Issues](https://github.com/richiehug/opi-dotnet-sdk/issues) for bugs and feature requests. Support is best effort, with no SLA or guaranteed resolution.

## License

The compiled SDK is distributed under the [OPI .NET SDK Binary License](LICENSE). Commercial application integration and bundling of the unmodified SDK are permitted. Source code is not included or licensed for redistribution. This is not an open-source license.

OPI .NET SDK is an independent software project created and maintained by [Richard Hug](https://richiehug.com). This SDK is not owned, maintained, supported, warranted or endorsed by payment providers. It follows the OPI protocol specifications.

Company references describe compatibility and tested environments only. There is no SLA, guaranteed response or resolution time, release schedule, or commitment to resolve individual issues. Its goal is simple: make the OPI protocol brilliantly straightforward to use from .NET.
