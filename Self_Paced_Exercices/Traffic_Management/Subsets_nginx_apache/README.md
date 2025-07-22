
Create the Deployment, Service and Gateway.

Test with a curl against the LB to test if it works/replies.

Add a Virtual Service with 3 subsets (single destination rule).

Default uses the created service, as a backend.

Path /nginx routes to the same backend but using the Nginx subset
Path /apache routes to the same backend but using the Apache subset

/apache has to only be resolved by apache, /nginx with nginx, default can iterate.
