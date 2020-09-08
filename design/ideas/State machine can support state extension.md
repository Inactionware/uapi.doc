## Propsal

Currently the state machine can be combined to a service to provide state management function, but the state can not be extend, sometime we want to create new service extends existing service and reuse existing state machine, but new service need add new state to existing state managment.

## Solution

Initial idea is make state shifter to a chain, the extensional state shifter will be matched and executed first, if no extensional state shifter is matched then parent state shifter will be matched and executed, if no any state shifter is matched the requested state, then an exception will be risen.