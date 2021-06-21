# SessionState

When a function called from a module runs this code

`$PSCmdlet` will include variables from the parents scope while `$ExecutionContext` will contain a fresh scope.

So if they function calling has `$VerbosePreference` set, it will not automatically propagate to the callee. We can retrieve the callers preference through `$PSCmdlet`

```powershell
$PSCmdlet.SessionState.PSVariable.Get("VerbosePreference") | Out-Host
$ExecutionContext.SessionState.PSVariable.Get("VerbosePreference") | Out-Host
```