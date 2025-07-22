# Description

This section will focus on the interaction with the backend and routing the traffic towards it.

## Examples

01-Service_Entry
02-HTTPS-backend
03-Outboud-Traffic-Policy
04-HTTPS-backend-with-mTLS (TODO)

## Heads up

On the example `02-Outboud-Traffic-Policy`, Istio's `meshConfig.outboundTrafficPolicy` will require to be modified.

On the example it's used the `istioctl install` command to set that up, as I assume you are testing this examples in a sandbox that you are free to "destroy".

