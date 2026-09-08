# Nexi .NET SDK

Bring Nexi payments straight into your cashier app. 🎉✨

The Nexi .NET SDK brings your cashier app and the Nexi EP2 payment terminal together in one seamless payment experience. Its modern C# API makes it easy to start transactions, work with receipts, and manage the terminal without leaving your app. It simplifies integrating your ECR app with Nexi Switzerland.

## Full and Lite

| Edition | Compatibility | Presentation |
| --- | --- | --- |
| **Full** — `nexi-dotnet-sdk-<version>.zip` | Windows, .NET 10 (`net10.0-windows`) with WPF support | Optional SDK overlay included in `Nexi.Sdk.dll` |
| **Lite** — `nexi-dotnet-sdk-lite-<version>.zip` | Windows, .NET Framework 4.8 / 4.8.1 (`net48`) | No SDK overlay; use your application's UI |

Use Full for modern Windows integrations. Lite provides backward compatibility for existing .NET Framework 4.8 cashier applications. Both editions expose the same payment, receipt, terminal-operation and recovery API. Choose one distribution; do not combine their DLLs. Lite includes the Microsoft dependency DLLs required by Framework 4.8.

Full and Lite use version **1.0.0** and are available from the [GitHub release](https://github.com/richiehug/nexi-dotnet-sdk/releases/tag/1.0.0).

## Quick start

First, [add the local SDK DLL to your application](integration.md#add-the-sdk-to-an-application). Then create an instance for your network terminal and take your first payment:

```csharp
using Nexi.Sdk;

await using var nexiSdk = new NexiClient(new NexiOptions
{
    TerminalHost = "192.168.0.69",
    WorkstationId = "CASHIER-01"
});

async Task TakePaymentAsync()
{
    var result = await nexiSdk.PaymentAsync(
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
- Show an optional Windows transaction overlay with a configurable Abort button (**Full only**), or provide your own UI in either edition.
- Recover uncertain transaction results without automatically resending the payment.
- Diagnose integration issues with optional file logging that excludes raw card and receipt data.

## Documentation

- [Integration guide and API overview](integration.md)
- [Detailed C# API reference](https://richiehug.github.io/nexi-dotnet-sdk/)

The integration guide covers installation, configuration, all public terminal methods, request and response models, result states, errors, receipts, recovery, and operation events. XML IntelliSense documentation is included with the DLLs.

## Requirements

- Full: a Windows .NET 10 application with WPF support
- Lite: a Windows .NET Framework 4.8 or 4.8.1 application; no WPF dependency in the SDK
- Asynchronous C# support; see the integration guide for older C# syntax
- A provisioned Nexi EP2 payment application reachable through the terminal's network address

## Support

For help, questions, or bug reports, please [open an issue in this repository](https://github.com/richiehug/nexi-dotnet-sdk/issues).
