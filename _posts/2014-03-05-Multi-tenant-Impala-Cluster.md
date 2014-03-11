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


# Understanding communication with HS2 server

The [Cloudera HiveServer](https://github.com/pauldeschacht/hive/tree/cdh4.5.0-release/service/src/java/org/apache/hive/service) starts 2 thrift services: the CLIServer and the ThriftCLIService. This post will concentrate on the ThriftCLIService, as it implements the HiveServer2 protocol. As mentioned in the [Thrift post](pauldeschacht.github.io/thrift/2014/02/27/Understanding-Trift.html#additional_information), the service binds a TTransport and a TProcessor together. The type of transport (TServerSocker, TSSLTransport, ...) is determined by the hive-site parameters file and is handled by the [HiveAuthFactory](https://github.com/pauldeschacht/hive/blob/cdh4.5.0-release/service/src/java/org/apache/hive/service/auth/HiveAuthFactory.java#L118).

The possible options are NONE, SASL, LDAP, KERBEROS and CUSTOM. Since the authentication is granted by an external REST server.
