# Integrating the Nexi .NET SDK

This guide covers installation, configuration, terminal operations, result handling, recovery, and diagnostics. For platform requirements, see the [README](README.md). The [generated C# API reference](https://richiehug.github.io/nexi-dotnet-sdk/) lists every public type, member and overload after publication.

## Add the SDK to an application

### Local DLL

Choose **Full** for Windows .NET 10 (`net10.0-windows`, WPF support) or **Lite** for backward compatibility with .NET Framework 4.8/4.8.1 (`net48`). Both expose the same `Nexi.Sdk` payment API. Full includes the optional Windows overlay in the same DLL; Lite has no SDK UI.

Copy `Nexi.Sdk.dll` and its XML IntelliSense file from the chosen ZIP to `libs/`. For Full, add this reference:

```xml
<ItemGroup>
  <Reference Include="Nexi.Sdk">
    <HintPath>libs/Nexi.Sdk.dll</HintPath>
    <Private>true</Private>
  </Reference>
</ItemGroup>
```

For **Lite**, copy **all** DLLs from its ZIP, including the Microsoft dependencies. Reference the complete set instead of the single Full reference:

```xml
<PropertyGroup>
  <TargetFramework>net48</TargetFramework>
  <AutoGenerateBindingRedirects>true</AutoGenerateBindingRedirects>
  <GenerateBindingRedirectsOutputType>true</GenerateBindingRedirectsOutputType>
</PropertyGroup>
<ItemGroup>
  <Reference Include="libs/*.dll">
    <Private>true</Private>
  </Reference>
</ItemGroup>
```

For older project formats, use **Add Reference → Browse** to select those DLLs and enable automatic binding redirects in the executable project. Deploy the generated application `.config` file as well. Test with the cashier's existing dependency versions; do not overwrite its Microsoft DLLs blindly.

Full uses the Windows Desktop runtime; Lite uses the installed .NET Framework 4.8/4.8.1 runtime and its bundled dependencies. Distribution remains DLL-only; integrators do not need a Nexi NuGet package. Do not reference Full and Lite together. If upgrading from the initial two-DLL download, remove the separate `Nexi.Sdk.Windows.dll` reference and use the new Full `Nexi.Sdk.dll` instead.

## Initialize the SDK

```csharp
using Nexi.Sdk;

var options = new NexiOptions
{
    TerminalHost = "192.168.0.69",
    WorkstationId = "CASHIER-01",
    PayChannelPort = 4100,
    DeviceChannelPort = 4102,
    Language = "de",
    DataDirectory = Path.Combine(
        Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData),
        "YourCashier", "Nexi"),
    Receipts = new ReceiptHandling
    {
        MerchantReceipt = ReceiptHandlingMode.Available,
        CustomerReceipt = ReceiptHandlingMode.Available
    }
};

await using var terminal = new NexiClient(options);
```

The examples use modern C# syntax. Older C# 7.3 applications can use ordinary object initializers for `NexiOptions` and `ReceiptHandling`; `required` and `init` are not required to configure the SDK. Replace `await using` with explicit asynchronous cleanup:

```csharp
var terminal = new NexiClient(options);
try
{
    var result = await terminal.PaymentAsync(1250, "CHF");
    // Handle and persist result, including both receipt copies.
}
finally
{
    await terminal.DisposeAsync();
}
```

In a real cashier retain the client for the terminal's lifetime and dispose it only after outstanding operations finish. `CloseAsync()` is the terminal close-day operation, not client disposal. A .NET Framework 4.8/C# 7.3 consumer is compiled as part of SDK checks.

Retain one instance per terminal. Use a stable workstation ID and storage directory across application restarts. The terminal's callback configuration must point to the cashier machine. Configure Windows Firewall to allow callbacks from the terminal on the device port. A callback listener is created internally during each exchange, and callback peers are restricted to the resolved terminal address. This is raw OPI TCP on a trusted shop network; it is not an internet endpoint.

Do not create competing instances with different host aliases or data directories for the same terminal. Multiple terminals on a single cashier need explicitly coordinated, distinct callback ports and matching terminal configuration. This initial SDK is intended for one owner per terminal.

## Optional Windows overlay

**Full only:** `EmbeddedTransactionOverlay` and `WindowsTransactionOverlay` are included in the Full `Nexi.Sdk.dll`, under the existing `Nexi.Sdk.Windows` namespace. No second SDK DLL is needed. Full hosts must include WPF support, including WinForms applications:

```xml
<PropertyGroup>
  <TargetFramework>net10.0-windows</TargetFramework>
  <UseWPF>true</UseWPF>
</PropertyGroup>
```

For WPF, host the SDK overlay **inside the cashier window**. Put an empty overlay Grid above a sibling containing the application's content:

```xml
<Grid>
    <Grid x:Name="CashierContent">
        <!-- Cashier UI -->
    </Grid>
    <Grid x:Name="SdkOverlayHost" Panel.ZIndex="50" />
</Grid>
```

Create the adapter on the UI thread, then pass it to the core SDK:

```csharp
using Nexi.Sdk;
using Nexi.Sdk.Windows;

await using var overlay = new EmbeddedTransactionOverlay(
    SdkOverlayHost, CashierContent, options.AbortRetryDelay);
await using var terminal = new NexiClient(options with
{
    Overlay = overlay,
    ShowAbortButton = true
});
var result = await terminal.PaymentWithRecoveryAsync(1250, "CHF", "SALE-12345");
```

`EmbeddedTransactionOverlay` covers and blocks only the specified cashier content. It blurs that content in normal rendering mode and uses dimming without a live blur in software-only mode to keep animation responsive. It creates no separate window and restores the previous enabled/effect state after a 200 ms fade on completion. The cashier remains blocked until that dismissal finishes. Keep the overlay host outside the cashier content, spanning the full client area. The app should also block window closure during active operations. Await SDK calls; do not block the UI dispatcher with `.Result` or `.Wait()`.

For WinForms or applications that prefer a separate dialog, `WindowsTransactionOverlay(cashierWindowHandle)` remains available with the same visual design on its own UI thread. Pass the WPF `WindowInteropHelper` handle or WinForms `Form.Handle`. The owner is re-enabled before the overlay closes. Omitting the handle requires the host to manage its own interaction blocking.

Omit `Overlay` to use only the SDK API and events. **Lite does not contain the Windows overlay classes.** Both editions retain the lightweight `ITransactionOverlay` interface for app-owned presentation adapters. The Framework 4.8 demo supplies its own WPF overlay with the same visual design. `OnEvent` must enqueue work without blocking; exceptions from presentation callbacks are isolated. A failure to create the overlay prevents the operation from starting. Dispose the SDK before disposing its overlay.

## Payments

```csharp
var payment = await terminal.PaymentAsync(1250, "CHF");
```

Amounts are positive integer minor units. CHF 12.50 is `1250`. Currency codes are normalized and validated; conversion also supports zero-, three- and four-decimal currencies. Merchant and authorization references are separate. References accept 1–20 ASCII letters, digits, underscores or hyphens. Reverse-last has no amount or reference and can only reverse the terminal's last eligible transaction.

An ordinary operational failure is returned as `NexiResult`. Invalid SDK configuration, unavailable/corrupt storage during construction and use after disposal can throw. Persist each financial result against its sale using `CorrelationId`. A final `Amount` includes any positive `Tip` reported by the terminal.

## Refunds

```csharp
// Unreferenced refund; no original authorization identifier is needed.
var refund = await terminal.RefundAsync(1250, "CHF");
// Referenced refund, when supported and the original authorization is available.
var referencedRefund = await terminal.RefundAsync(1250, "CHF", authReference: "605224");
```

Use a positive amount in minor units. The demo application always uses unreferenced refunds; the SDK supports both forms.

## Reversals

```csharp
var reversal = await terminal.ReversalAsync();
```

This reverses the terminal's last eligible transaction. It does not accept an amount or an arbitrary transaction identifier.

## Handle results

```csharp
if (payment.IsSuccessful)
{
    // Complete the sale and store payment.Transaction and its receipts.
}
else if (payment.Status == ResultState.Unknown)
{
    // Keep the sale unresolved. Do not retry the payment automatically.
}
else
{
    // Display the outcome using Status, Error and ErrorCode.
}
```

### Payment method

`Transaction.PaymentMethod` is the terminal's brand or payment-method label. It is optional; do not infer approval from its presence. Use `Status` to decide the outcome.

### Tip results

`Transaction.Amount` includes a positive `Transaction.Tip` reported by the terminal. Both use minor units. Do not add the tip a second time.

### DCC results

`Transaction.Dcc` optionally contains `Offered`, `Accepted`, `Amount`, `Currency`, `CurrencyCode`, `ExchangeRate` and `MarkupPercentage`. Preserve the terminal's strings for display; do not recalculate settlement totals from them.

## Receipt handling

```csharp
string? merchant = payment.Transaction?.Receipts?.MerchantReceipt?.Content;
string? customer = payment.Transaction?.Receipts?.CustomerReceipt?.Content;
```

`PrintLocal` means the payment terminal prints. `Available` means printable text is returned to the cashier. `Unavailable` suppresses the requested copy. Each copy is configured independently. Missing copies are normal if the terminal did not generate them or the mode did not request their return.

Receipt content is plain text, with line breaks and leading alignment spaces preserved. Render it in a monospace font. The cashier owns printing and durable storage. A receipt-only reprint has `Transaction.Receipts` but may have no amount or payment details.

## Events and cancellation

```csharp
terminal.OperationEvent += (_, e) =>
{
    if (e.Kind == OperationEventKind.TerminalMessage)
    {
        // Post e.Message to the cashier's UI thread.
    }
    if (e.Kind == OperationEventKind.TransactionRecovered)
    {
        // Update the ORIGINAL sale using e.Result.CorrelationId.
    }
    if (e.Kind == OperationEventKind.ReceiptCaptured)
    {
        // Especially useful for receipts recovered by a reprint operation.
        // Keep e.Operation and e.CorrelationId; do not attach an old ticket to a new sale.
    }
};

var abort = await terminal.AbortAsync();
```

Events are delivered in order on a background worker. Slow or throwing application event handlers do not delay device acknowledgements. Events may arrive just after the awaited result; the result already contains its final receipts. Do not synchronously wait for SDK disposal from an event handler.

Abort bypasses the regular operation lock and uses a separate control connection. Attempts are delayed and limited. An abort response does not override an approved payment. If the abort response is lost, the abort call returns uncertainty and the financial operation continues resolving its own result. Do not cancel a local task and assume the payment was cancelled.

## Uncertain results and restart recovery

Before transmission the SDK durably saves the request identity, correlation ID, amount, currency and reference. It stores no card data in this record. If the final response is lost, timed out, malformed, or otherwise uncertain, the SDK attempts `RepeatLastMessage` and matches both original request ID and request type. It never resends that financial request.

```csharp
var recovered = await terminal.ReconcilePendingAsync();
if (recovered?.Status == ResultState.Unknown)
{
    // Keep the sale unresolved and reconcile with terminal/acquirer records.
}
```

Call this at startup before enabling new payments. A mismatched last transaction remains `Unknown` with `RecoveryMismatch`; it does not prove the original payment failed. Pending metadata is retained. Manual reconciliation is required if the terminal can no longer provide the original result. There is intentionally no automatic discard of unresolved metadata.

If a new financial call encounters pending metadata, it only reconciles the old request and returns `InvalidRequest` for the new call with `PREVIOUS_TRANSACTION_RECOVERED` or `PENDING_TRANSACTION_UNRESOLVED`. The old result is delivered through `TransactionRecovered` when resolved. It is never returned as approval for the new sale. After handling the old sale, the cashier can explicitly start a new payment.

`PaymentWithRecoveryAsync` is an alternative to `PaymentAsync`. Only after an explicit `PrintLastTicket` rejection does it wait `ReceiptRecoveryDelay`, reprint the previous ticket, verify readiness, and try the payment once. A failed reprint stops this flow: the original payment rejection is returned and no new payment is sent. The reprint failure appears in recovery events and diagnostics. It never retries an uncertain payment. Receipts from this reprint belong to the previous transaction and are delivered as reprint events.

Defaults: connect timeout 10 seconds, exchange timeout 5 minutes, network-unavailable grace 5 seconds, result recovery delays 1/2.5/5/10/20 seconds. Recovery exchanges each have their own operation timeout, so the whole call can outlast a single exchange timeout. Network availability is OS-level; a silent terminal on an otherwise working network is detected by socket errors or the exchange timeout.


### Recover the previous sale, then start a new one

Use two awaited calls so the two results stay separate:

```csharp
var previous = await terminal.ReconcilePendingAsync();
if (previous != null)
{
    // Update the OLD sale, using previous.CorrelationId or
    // previous.Transaction?.Reference. Make this update idempotent.
    SaveRecoveredSale(previous);

    if (previous.Status is ResultState.Unknown
        or ResultState.InProgress
        or ResultState.CommunicationError)
        return; // Keep the old sale unresolved; do not start another payment.
}

// Persist this new sale/reference before calling the terminal.
var current = await terminal.PaymentWithRecoveryAsync(
    amount: 1250,
    currency: "CHF",
    reference: "SALE-12346");
SaveNewSaleResult(current);
```

`SaveRecoveredSale` and `SaveNewSaleResult` represent the integrator's own storage methods. The recovered result keeps the original correlation ID; the new payment has its own ID. Store a unique merchant reference before sending each payment so a recovered result can be matched even if the cashier closed before saving the first result. A null recovery result means there is no SDK pending record, not that an independently saved cashier intent failed.

Recovery also emits `TransactionRecovered`. If you handle both that event and the returned result, use an idempotent update to avoid duplicate history. If a payment call was made before explicit recovery, handle `TransactionRecovered` for the old sale and inspect the new call's `PREVIOUS_TRANSACTION_RECOVERED` or `PENDING_TRANSACTION_UNRESOLVED` error code; a new payment was not sent.

`PaymentWithRecoveryAsync` returns one result for the payment being requested. Its pending-ticket helper may emit reprint receipts from an older transaction, but does not return a pair of old/new financial results. Use `ReconcilePendingAsync` for the previous unfinished financial outcome. Returned receipt copies, when present, are in each result's `Transaction.Receipts`.

## File logging and deployment

`LoggingEnabled = true` enables bounded daily files in `DataDirectory/logs`. Logs include configured ports/timeouts, connection stages, socket error codes, request/response byte counts, elapsed times, lifecycle events, receipt type/length, outcome/error codes and numbered recovery attempts. Raw XML, terminal-message text, receipt text and card fields are excluded. `Logger` optionally receives those same diagnostics. Diagnostic writes run on the background event queue rather than delaying callback acknowledgements; dispose the SDK to flush queued entries. A logging failure does not change a payment result.

Deploy the selected `Nexi.Sdk.dll` alongside the cashier. Lite also needs its dependency DLLs and the host's binding-redirect configuration; .NET Framework 4.8/4.8.1 must be installed. A Full .NET 10 cashier can be published self-contained. No separate OPI Proxy process is needed.

## API overview

Namespace: `Nexi.Sdk`. Optional Windows UI namespace: `Nexi.Sdk.Windows`.

## Terminal operations

All terminal calls return `Task<NexiResult>` unless indicated otherwise.

| Method | Purpose |
| --- | --- |
| `PaymentAsync(amount, currency, reference?)` / `PaymentAsync(PaymentRequest)` | Payment |
| `PaymentWithRecoveryAsync(...)` | Payment with explicit pending-ticket recovery |
| `RefundAsync(amount, currency, authReference?)` | Referenced or unreferenced refund |
| `ReversalAsync()` | Reverse last eligible transaction |
| `AbortAsync()` | Out-of-band abort of active payment/refund |
| `ActivateAsync()` / `DeactivateAsync()` | Activate/deactivate terminal |
| `LoginAsync()` / `LogoffAsync()` | OPI login/logoff variants |
| `InfoAsync()` | GetInfo |
| `StatusAsync()` / `TerminalInfoAsync()` | GetStatus |
| `ReprintAsync()` | TicketReprint |
| `SubmitAsync()` | TransmitTrx |
| `CloseAsync()` | CloseDay |
| `ConfigAsync()` | ContactTMS |
| `InitAsync()` | ContactAcq |
| `ResetAsync()` | RestartTerminal |
| `RepeatLastMessageAsync()` | Read the last financial response; strictly reconcile SDK pending metadata when present |
| `ReconcilePendingAsync()` | `Task<NexiResult?>`; recover an earlier uncertain transaction |
| `DisposeAsync()` | `ValueTask`; release SDK ownership after operations |

## Result and event models

`NexiResult` contains `Status`, `Error`, `ErrorCode`, `CorrelationId`, optional `Transaction`, optional `Terminal` and `IsSuccessful`.

`TransactionResult` contains `Reference`, `Amount`, `Currency`, `Tip`, `PaymentMethod`, `MaskedCardNumber`, `AuthReference`, `ApprovalCode`, `TransactionDate`, `AcquirerId`, `Receipts`, and `Dcc`. Fields are optional and depend on terminal evidence. `ReceiptDetails` separates `MerchantReceipt` and `CustomerReceipt`; `Receipt.Content` is plain text.

`OperationEvent` contains `Operation`, `CorrelationId`, `Kind`, and optional `Message`, `ReceiptType`, `Receipt`, and `Result`. Kinds include Started, Connected, TerminalMessage, ReceiptCaptured, DccCaptured, RecoveryStarted, RecoveryCompleted, TransactionRecovered and Completed.

`NexiOptions` contains the client configuration. The client snapshots these settings when constructed; changing the options afterwards does not reconfigure an existing client. Supply `TerminalHost` and `WorkstationId`. Optional settings cover ports, language, receipts, force acceptance, PAN request preference, timeouts, recovery, data directory, logging and presentation. See configuration defaults below and the generated API reference for the full property list.

## Optional overlay

`ITransactionOverlay` has `ShowAsync(operation, language, abort?)`, non-blocking `OnEvent(event)`, and `CloseAsync()`.

**Full only:** `EmbeddedTransactionOverlay(hostGrid, cashierContent, abortRetryDelay?)` and `WindowsTransactionOverlay(ownerHandle?)` implement the adapter and `IAsyncDisposable`. Lite supports app-owned adapters through the same interface.

## Result states

| State | Meaning |
| --- | --- |
| `Success` | Operation completed successfully |
| `Declined` | Terminal/acquirer decline evidence |
| `Aborted` | Completed cancellation |
| `CommunicationError` | Connection or exchange problem; examine the error and operation context |
| `TerminalError` | Terminal or SDK operational error |
| `InvalidRequest` | Invalid input or a request blocked before execution |
| `InProgress` | Terminal is busy or still processing |
| `Unknown` | No reliable financial outcome; reconcile before another attempt |

`NexiError` provides stable categories: Failure, Aborted, ReprintRequired, DeviceUnavailable, TerminalBusy, LoginRequired, ResultUnknown, RecoveryMismatch, ConnectionError, OperationTimeout, InvalidRequest, InvalidOpiResponse, StorageError and Other. `ErrorCode` preserves a more specific terminal or SDK reason.

## Configuration defaults

| Option | Default |
| --- | --- |
| `TerminalHost`, `WorkstationId` | Required |
| `PayChannelPort`, `DeviceChannelPort` | `4100`, `4102` |
| `Language` | `en`; also supports `de`, `fr`, `it` |
| Merchant/customer receipt mode | `PrintLocal` for both |
| `ForceAcceptance` | `false` |
| `RequestFullPan` | `null` (no explicit preference) |
| `ConnectTimeout` | 10 seconds |
| `OperationTimeout` | 5 minutes per exchange |
| `ConnectivityLossGracePeriod` | 5 seconds |
| `RecoveryDelays` | 1, 2.5, 5, 10, 20 seconds |
| `AbortRetryDelay`, `MaxAbortAttempts` | 1.5 seconds, 3 |
| `AbortRecoveryDelay`, `AbortRecoveryAttempts` | 350 milliseconds, 3 |
| `ReprintAfterAbort`, `ReceiptRecoveryDelay` | `true`, 2 seconds |
| `DataDirectory` | Local application data / Nexi / Sdk |
| `LoggingEnabled`, `MaxLogStorageBytes` | `false`, 20 MiB |
| `Logger`, `Overlay` | `null` |
| `ShowAbortButton` | `true` |

The API website is generated from the SDK projects and XML comments during each SDK build. Publishing a release copies the matching version into the public website and updates its landing page. New public signatures therefore appear automatically; maintainers update XML comments when behavior changes.

## Repeat Last Message

```csharp
var last = await terminal.RepeatLastMessageAsync();
```

This sends OPI `RepeatLastMessage`, never a new payment. With an SDK pending record it applies the same request-ID and request-type checks as `ReconcilePendingAsync`. Without a pending record it returns the terminal's original financial outcome for inspection; its correlation ID belongs to this lookup. Do not automatically associate that response with a saved sale. Missing or unsupported original financial headers return `Unknown` rather than treating the outer response's success as payment approval.

`ReconcilePendingAsync` shows the configured overlay while recovery runs, without an Abort button. A null result means there is no recovery record for this SDK configuration; it does not prove that a separately saved cashier intent failed. Keep the original terminal/workstation settings and storage when restarting.

## Storage locations

The host application sets `NexiOptions.DataDirectory`. The default is the current user's local application data directory followed by `Nexi/Sdk`. Recovery metadata is scoped to terminal host and payment port beneath that directory. Daily diagnostics are in `DataDirectory/logs` when `LoggingEnabled` is true. The SDK does not save a cashier transaction-history database: the application owns transaction and receipt persistence.

The Windows demo chooses `%LOCALAPPDATA%/Nexi/DemoWin` in `App.xaml.cs` and passes its `sdk` subfolder to the SDK through `DemoSettings.SdkOptions`. Its `transactions` subfolder holds saved attempts/results and receipts; `recovered-receipts` holds separately recovered tickets. Renaming the app does not change these locations or migrate/delete existing data.

Repeat Last Message omits the receipt-header request flag, matching the proxy. A successful repeat envelope with original `PrintLastTicket` maps to `TerminalError` / `ReprintRequired`; a matching pending record is resolved without sending another payment. Diagnostics include outer/original result codes and request IDs/types for troubleshooting, without raw payloads.
