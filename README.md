# OPI .NET SDK

A .NET SDK for integrating OPI-compatible payment terminals into Windows POS and ECR applications.

> **Independent project**
>
> OPI .NET SDK is an independent software project created and maintained by [Richard Hug](https://richiehug.com). It is not an official Nexi product and is not owned, maintained, supported, warranted or endorsed by Nexi.
>
> Company references describe compatibility and tested environments only. This lightweight integration library is maintained on a best-effort basis, allowing fixes and compatibility improvements to be released quickly. There is no SLA, guaranteed response or resolution time, release schedule, or commitment to resolve individual issues.

## Full and Lite

| Edition | Compatibility | Presentation |
| --- | --- | --- |
| **Full** — `opi-dotnet-sdk-<version>.zip` | Windows, .NET 10 (`net10.0-windows`) with WPF support | Optional SDK overlay included in `Opi.Sdk.dll` |
| **Lite** — `opi-dotnet-sdk-lite-<version>.zip` | Windows, .NET Framework 4.8 / 4.8.1 (`net48`) | No SDK overlay; use your application's UI |

Use Full for modern Windows integrations. Lite provides backward compatibility for existing .NET Framework 4.8 cashier applications. Both editions expose the same payment, receipt, terminal-operation and recovery API. Choose one distribution; do not combine their DLLs. Lite includes the Microsoft dependency DLLs required by Framework 4.8.

Full and Lite use version **1.0.0** and are available from the [GitHub release](https://github.com/richiehug/opi-dotnet-sdk/releases/tag/1.0.0).

## Quick start

First, [add the local SDK DLL to your application](integration.md#add-the-sdk-to-an-application). Then create an instance for your network terminal and take your first payment:

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
- Show an optional Windows transaction overlay with a configurable Abort button (**Full only**), or provide your own UI in either edition.
- Recover uncertain transaction results without automatically resending the payment.
- Diagnose integration issues with optional file logging that excludes raw card and receipt data.

## Compatibility

Development and real-world validation have primarily been performed with OPI-compatible payment terminals used with Nexi Switzerland. Validate the specific terminal model, payment application, firmware and provider before production rollout. Compatibility with other OPI environments has not been established by this refactor.

## Documentation

- [Integration guide and API overview](integration.md)
- [Detailed C# API reference](https://richiehug.github.io/opi-dotnet-sdk/)

The integration guide covers installation, configuration, all public terminal methods, request and response models, result states, errors, receipts, recovery, and operation events. XML IntelliSense documentation is included with the DLLs.

## Requirements

- Full: a Windows .NET 10 application with WPF support
- Lite: a Windows .NET Framework 4.8 or 4.8.1 application; no WPF dependency in the SDK
- Asynchronous C# support; see the integration guide for older C# syntax
- A provisioned OPI-compatible payment application reachable through the terminal's network address

## Support

Use [GitHub Issues](https://github.com/richiehug/opi-dotnet-sdk/issues) for bugs and feature requests. Support is best effort, with no SLA or guaranteed resolution. See [SUPPORT.md](SUPPORT.md).

## Security

Report vulnerabilities privately; do not post payment data, credentials or sensitive logs in public issues. See [SECURITY.md](SECURITY.md).

## License

The compiled SDK is distributed under the [OPI .NET SDK Binary License](LICENSE). Commercial application integration and bundling of the unmodified SDK are permitted. Source code is not included or licensed for redistribution. This is not an open-source license.

## Maintainer

Richard Hug — [richiehug.com](https://richiehug.com).

## Overlay branding

Full's window and embedded overlays accept an inline SVG logo, header/button color and white or black text through `OverlayBrandingOptions`. Omit branding for the generic OPI design. Invalid or unsupported SVG falls back safely. See [custom branding and example code](integration.md#custom-overlay-branding-full-only).
