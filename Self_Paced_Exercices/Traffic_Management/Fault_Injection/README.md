

Create a VirtualService that routes the traffic towards the helloworld deployment.

Make that a 30% of that traffic gets the connection aborted.

50% of the traffic should receive a 4 second delay.


```shell
kubectl create -f ./src
```