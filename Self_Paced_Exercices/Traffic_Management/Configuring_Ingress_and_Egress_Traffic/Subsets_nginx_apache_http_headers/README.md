Same as the non-headers version.

Instead of selecting the backend/subset by path, select by the specified header.

Traffic with header `backend=nginx` goes to nginx, `backend=apache` goes to apache, if the header `backend` is specified but didn't match a previous rule return a 403 status code.

If no header `backend` is specified, send to the SVC created that will iterate between both possible backends.