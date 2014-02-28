---
layout: post
title:  "Multi tenant Impala Cluster"
date:   2014-03-05
categories: impala 
---

The goal is to create a *multi tenant Impala cluster*. Probably, today Cloudera Sensy can play this role. The project (open sourced at http://github.com/pauldeschacht/cloudera-hive) is a server based on HiveServer2. This server sits between the external clients (who uses ODBC) and the actual Impala cluster. It will verify each SQL statement. The verification itself is done by an authorization server. If the authorization server validates the acces, the intermediate server will simply forward the request to the actual Impala cluster.

![Infrastructure multi tenant Impala cluster](/images/infra_impala.gif)

The intermediate server is nothing more than a proxy:
1. Capture each SQL statement and verify with the authorization service.
2. If the access is authorized, forward the query to the Impala cluster.



