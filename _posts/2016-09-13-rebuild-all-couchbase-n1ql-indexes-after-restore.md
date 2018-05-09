---
id: 207
title: Rebuild All Couchbase N1QL Indexes After Restore
date: 2016-09-13T14:39:36+00:00
guid: http://btburnett.com/?p=207
permalink: /2016/09/rebuild-all-couchbase-n1ql-indexes-after-restore.html
categories:
  - Couchbase
---
When restoring a Couchbase cluster from a backup, the restore utility is kind enough to recreate the N1QL indexes for you.  To improve speed and efficiency, the indexes are only created, they are not built automatically.  Before they can be used, you must execute a build command such as this:

<pre class="brush: sql; title: ; notranslate" title="">BUILD INDEX ON BucketName (IndexName1, IndexName2, IndexName3)
</pre>

It is important that this query be issued as a single command for all indexes on a bucket.  This allows the indexes to be built together, resulting in only one read of the data from the cluster while building multiple indexes.

## The Problem

Unfortunately, N1QL doesn&#8217;t currently offer a wildcard option, so there is no quick way to rebuild all indexes without typing the all of their names.  If you&#8217;re trying to script your environments for development or QA this can be particularly problematic, as the list of indexes may not be constant. It could also be a problem when creating scripts for a disaster recovery plan.

## The Solution

If you&#8217;re running on Linux (you should be for production clusters), the solution is to use this script:

<pre class="brush: bash; title: ; notranslate" title="">#!/bin/sh

QUERY_HOST=http://localhost:8091

for i in "BucketName1" "BucketName2" "BucketName3"
do
  /opt/couchbase/bin/cbq -e $QUERY_HOST -s="$( \
    echo "BUILD INDEX ON $i (\`$( \
      /opt/couchbase/bin/cbq -e $QUERY_HOST -q=true -s="SELECT name FROM system:indexes where keyspace_id = '$i' AND state = 'deferred'" | \
        sed -n -e '/{/,$p' | \
        jq -r '.results[].name' | \
        sed ':a;/.*/{N;s/\n/\`,\`/;ba}')\`)")"

  # Wait for completion
  until [ `/opt/couchbase/bin/cbq -e $QUERY_HOST -q=true -s="SELECT COUNT(*) as unbuilt FROM system:indexes WHERE keyspace_id = '$i' AND state &lt;&gt; 'online'" | sed -n -e '/{/,$p' | jq -r '.results[].unbuilt'` -eq 0 ];
  do
    sleep 5
  done
done
</pre>

Replace the QUERY_HOST parameter as needed to point to a query node, and replace the BucketName values with the names of the buckets where indexes must be built.  It will process each bucket one at a time, waiting for the indexes to be built before continuing to the next bucket.

The only depedency is the [jq](https://stedolan.github.io/jq/) utility, which is a command line JSON parser. On Ubuntu, this can be installed via:

<pre class="brush: bash; title: ; notranslate" title="">sudo apt-get install -y jq
</pre>

The script isn&#8217;t pretty, but it gets the job done. Hopefully N1QL will get a wildcard BUILD INDEX command in the future.

**Note:** Revised 9/15/2016 to better strip header information from the query output before the JSON object. Previously it only stripped the first line, now it strips all lines until it encounters the curly brace starting the JSON result.