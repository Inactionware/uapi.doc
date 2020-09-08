## Proposal

There are many service is used one time, if service repo still reference it which will cause memory leak, in other case some service is only referenced by one service, if that service is removed but service repo still reference it then it cause memory leak also

## Solution

*   For instance service of prototype service, since it is wrapped in InstanceServiceHolder, so service repo can drop it when it is injected to host service.
*   For other service, how to identify the service is used only one time?