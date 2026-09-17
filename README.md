# CacheViewExpirationBehavior

A Blazor Web App that uses static SSR to validate the `CacheView` expiration
behavior described in [dotnet/aspnetcore#69120](https://github.com/dotnet/aspnetcore/issues/69120).

## Source

- Public repository: https://github.com/NanthiniMahalingam/CacheViewExpirationBehavior
- Immutable tested commit: https://github.com/NanthiniMahalingam/CacheViewExpirationBehavior/commit/b302f1c9370af906fcbb08287cc556ca56767507

To reproduce the tested source exactly:

```powershell
git clone https://github.com/NanthiniMahalingam/CacheViewExpirationBehavior.git
Set-Location CacheViewExpirationBehavior
git checkout --detach b302f1c9370af906fcbb08287cc556ca56767507
```

## Prerequisites

- .NET SDK `11.0.100-rc.1.26425.128` (specified by `CacheViewExpiration/global.json`)
- A browser
- Optional: Visual Studio 2026 Preview for IDE-based testing

Confirm that the pinned SDK is selected:

```powershell
Set-Location CacheViewExpiration
dotnet --version
```

Expected output:

```text
11.0.100-rc.1.26425.128
```

## Restore and build

Run these commands from the `CacheViewExpiration` directory:

```powershell
dotnet restore CacheViewExpiration.csproj
dotnet build CacheViewExpiration.csproj --configuration Debug --no-restore
```

The build succeeds with no errors and writes the application to
`bin/Debug/net11.0`.

## Run

```powershell
dotnet run --project CacheViewExpiration.csproj --configuration Debug --no-build --launch-profile http
```

Wait for the console to report that the application is listening, then open
http://localhost:5090. Keep the process running for the whole timed test. Stop it
with `Ctrl+C` when testing is complete.

The server time changes on every request. Each cached section displays a GUID and
the time its child component was initialized. An unchanged GUID means the cached
content was reused; a changed GUID means the section expired and was rendered
again.

## Timed expiration test

Use the displayed **Current server time**, rather than wall-clock estimates, and
record each refresh time and GUID. Timing starts with the first page load (`T+0`).
Allow a small margin around each expiration boundary.

### Pass 1: absolute and default expiration

1. At `T+0`, load the page and record every GUID and the displayed fixed
	deadlines.
2. At about `T+5 seconds`, refresh. All GUIDs should be unchanged.
3. Continue refreshing every 5 seconds. This keeps both sliding entries active
	because every access occurs within their 10-second idle window.
4. On the first refresh after `T+10 seconds`, **ExpiresAfter** should have a new
	GUID. Refresh immediately again; that new GUID should remain unchanged.
5. Compare **ExpiresOn** immediately before and on the first refresh after its
	displayed deadline. It should remain cached before the deadline and change
	after it.
6. On the first refresh after `T+30 seconds`, **Default expiration** should have a
	new GUID. **ExpiresSliding only** should also have a new GUID even though it was
	accessed within every 10-second window. This proves that its sliding window is
	bounded by the 30-second default absolute expiration.
7. **ExpiresAfter and ExpiresOn** should retain its original GUID after the
	10-second `ExpiresAfter` period because `ExpiresOn` has precedence. It should
	change on the first request after its displayed `ExpiresOn` deadline. Because
	the fixed deadline remains in the past, an immediate subsequent refresh must
	not establish a new future caching window.
8. Keep refreshing within each 10-second window through the first refresh after
	`T+120 seconds`. **ExpiresSliding** should stay cached before its explicit
	two-minute absolute lifetime and change after that lifetime is exceeded.

### Pass 2: sliding idle expiration

Restart the application to reset the process-local cache, then:

1. At `T+0`, load the page and record both sliding GUIDs.
2. Refresh at approximately `T+5`, `T+10`, and `T+15 seconds`. Both sliding GUIDs
	should remain unchanged.
3. Leave the page idle for more than 10 seconds; 12 seconds gives a useful timing
	margin.
4. Refresh once. Both sliding GUIDs should change on this first request after the
	idle window.
5. Refresh immediately. The new GUIDs should remain unchanged.

### Process lifetime check

1. Record all GUIDs, stop the app with `Ctrl+C`, and start it again with the same
	`dotnet run` command.
2. Reload http://localhost:5090.
3. Verify that every GUID differs from the values recorded before restart. The
	default cache store is process-local, so no entry should survive a restart.

## Expected result summary

| Section | Configuration | Expected expiration |
| --- | --- | --- |
| ExpiresAfter | 10-second period | First request after 10 seconds from entry creation |
| ExpiresOn | Fixed displayed deadline | First request after the deadline |
| ExpiresSliding | 10-second idle window and 2-minute absolute lifetime | After more than 10 seconds idle, or at the absolute limit |
| ExpiresSliding only | 10-second idle window | After more than 10 seconds idle or at the 30-second default absolute limit, whichever comes first |
| Default expiration | No expiration option | First request after 30 seconds |
| ExpiresAfter and ExpiresOn | 10-second period and fixed displayed deadline | `ExpiresOn` takes precedence; the 10-second period is ignored |

For evidence, retain a short table of server time and GUID per refresh. A GUID
should change only on the first request after its effective expiration boundary,
then remain stable on an immediate refresh.