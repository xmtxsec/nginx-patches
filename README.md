# Nginx 补丁

## dav-copy-hardlink

Implements hardlinking in DAV copy instead of real copy.

    dav_copy_hardlink on;

## per-worker-listener/new-per-worker-listener
Per-worker-listener is an old version and is not applicable to the latest nginx.

New-per-worker-listener is a patch that I used to solve monitoring exceptions when I was studying nginx-rtmp-module streaming. Applicable to nginx-1.21.1.

Implements per worker listeners to make requests to
certain nginx worker. This requires `accept_mutex off`;
Does not work on Windows.

    worker_processes 5;

    events {
        # required by per_worker
        accept_mutex off;
    }
    ...
    http {
        ...
        server {
            # usual listener
            listen 80;

            # per-worker listener
            # 1st worker will listen 8000
            # 2nd worker will listen 8001
            # 3rd worker will listen 8002
            # 4th worker will listen 8003
            # 5th worker will listen 8004
            listen 8000 per_worker;
            ...
        }
    }

## put-range

Implements HTTP `Content-Range` support in DAV PUT request header. When PUT'ting
large files it's now possible to split it into several requests and upload
the data simultaneously. The behavior is allowed by HTTP standard but not implemented
in nginx by default. However it's up to client to watch the consistency.

    location / {
        root /tmp;
        dav_methods PUT;
    }

Upload. Assume `/tmp/partX` are all 100-byte files.

    curl -X PUT -H "Content-Range: bytes 100-200/300" --data-binary @/tmp/part2 localhost:8088/testfile 
    curl -X PUT -H "Content-Range: bytes 200-300/300" --data-binary @/tmp/part3 localhost:8088/testfile 
    curl -X PUT -H "Content-Range: bytes 0-100/300" --data-binary @/tmp/part1 localhost:8088/testfile 

Download

    curl localhost:8088/testfile

## dav-put-report

Reports upload progress to the client as chunked response. Applied after `put-range`.
Report interval is read from X-Put-Report header.

    curl -XPUT -H "X-Put-Report: 500" -H "Content-Range: bytes 100-200/100" --data-binary @/tmp/test localhost:8000/dav/test
