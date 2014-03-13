---
layout: post
title:  "Multi tenant Impala Cluster"
date:   2014-03-05
categories: impala 
---

The goal is to create a *multi tenant Impala cluster*. In other words, a server that validates each SQL statement for each user. Probably, [Cloudera Sentry](http://www.cloudera.com/content/cloudera/en/products-and-services/cdh/sentry.html) can play this role and might replace this server. Nevertheless, the project allows to better understand the communication with Impala.

The [multi tenant server](http://github.com/pauldeschacht/cloudera-hive) is not directly based on the Impala server (C++) but on Cloudera's HiveServer2 (Java). Both Impala server and HiveServer2 implements the same [Thrift protocol](https://github.com/cloudera/Impala/blob/master/common/thrift/ImpalaService.thrift). This protocol is used in Cloudera's ODBC driver. It was a bit easier to modify the HiveServer2 code that the Impala code, therefore the multi tenant server is based on the [HiveServer2 implementation](https://github.com/cloudera/hive/tree/cdh4.5.0-release).

The multi tenant server sits between the external ODBC clients and the actual Impala cluster. It inspects each SQL statement. The verification itself is done by a remote authorization server. If the authorization server validates the acces, the intermediate server will simply forward the request to the actual Impala cluster.

![Infrastructure multi tenant Impala cluster](/images/infra_impala.gif)

The multi tenant server is nothing more than a proxy:
1. Capture each SQL statement and verify with the authorization service.
2. If the access is authorized, forward the query to the Impala cluster, otherwise return an error.

The [Cloudera Impala](https://github.com/cloudera/Impala) server launches [2](https://github.com/cloudera/Impala/blob/master/be/src/service/impala-server.cc#L1773) external facing Thrift services

* Impala/TCLIService Thrift service (aka Hive Server 2 protocol)  on port 21050, used by ODBC 2.5
* Beeswax Thrift service (aka Hive Server protocol) on port 21000, used by ODBC 1.2

The [Cloudera HiveServer2](https://github.com/pauldeschacht/hive/tree/cdh4.5.0-release/service/src/java/org/apache/hive/service) implements the  protocol.

* TCLIService Thrift service on port 1000

In order to create a multi tenant server, it is sufficient to implement the TCLIService. There are a couple of options:

* Start from scratch and implement the Thrift services. 
* Derive from Cloudera Impala server (but building Impala is not for the faint of heart)
* Derive from Cloudera HiveServer2

Although Java is not my prefered language, deriving from the HiveServer2 turned out to be fastest to get to a working prototype.

# Delegation of the Thrift commands

The idea is to intercept all the Thrift request (defined in the ImpalaService.thrift) and only take a specific action on 
* OpenSession
* CloseSession
* ExecuteStatement

All the other Thrift requests will simply be delegated to the actual HiveServer2. 

## Construction of the delegate

During initialization, the delegate is constructed. The delegate will forward the requests to the actual Impala server.

```Java
  @Override
  public synchronized void init(HiveConf hiveConf) {
    this.hiveConf = hiveConf;
    super.init(hiveConf);
    try {
        String host = HiveConf.getVar(hiveConf,IMPALA_SERVER_HOST);
        int port = HiveConf.getVar(hiveConf,IMPALA_SERVER_PORT)
        TSocket transport = new TSocket(host,port);
        transport.setTimeout(60000);
        TBinaryProtocol protocol = new TBinaryProtocol(transport);
        delegate = new TCLIService.Client(protocol);
        transport.open();
        LOG.info("ThriftCLIServiceDelegator connected to remove Impala daemon");
    }
    catch(Exception e) {
        LOG.error(e.toString());
    }
  }
```

## Forward the requests to the actual Impala server

For most Thrift requests, the request is simply forwarded to the actual Impala server.

```Java
  @Override
  public TGetTablesResp GetTables(TGetTablesReq req) throws TException {
      return delegate.GetTables(req);
  }
  @Override
  public TGetColumnsResp GetColumns(TGetColumnsReq req) throws TException {
      return delegate.GetColumns(req);
  }
```

## Capture the new sessions

The OpenSession request is first forwarded to the actual Impala server. The reply contains a session identifier which is then linked to the session's user information. A simple mapping allows to retrieve the user information from a session identifier.

```Java
  @Override
  public TOpenSessionResp OpenSession(TOpenSessionReq req) throws TException {
      TOpenSessionResp resp = delegate.OpenSession(req);
      String username = req.getUsername();
      String password = req.getPassword();
      String ip = null;
      if (hiveAuthFactory != null && hiveAuthFactory.getRemoteUser() != null) {
        username = hiveAuthFactory.getRemoteUser();
        ip = hiveAuthFactory.getIpAddress();
      } else {
        username = SessionManager.getUserName();
      }
      if(ip==null) {
          ip = getIpAddress();
      }
      if (username == null) {
        username = req.getUsername();
      }
      sessions.addSession(username,password,ip,resp.getSessionHandle());
      return resp;
```

## Capture the SQL statement 

Each request that executes a SQL statement is intercepted. The mapping from session to user information tells which user executes the SQL statement. The remote authentication server informs us whether the user has the permission to execute the statement. If granted, the request is forwarded to the actual Impala server. (TODO Otherwise)

```Java
  @Override
  public TExecuteStatementResp ExecuteStatement(TExecuteStatementReq req) throws TException {
      LOG.info("Execute statement : " + req.getStatement() );
      TSessionHandle sessionHandle = req.getSessionHandle();
      LocalSessionManager.SessionInfo sessionInfo = sessions.getSession(sessionHandle);
      if(sessionInfo != null) {
          LOG.info("User " + sessionInfo.username + " at " + sessionInfo.ip);
      }
      if( validateUserStatement(sessionInfo.username,sessionInfo.ip,req.getStatement() == true) {
        return delegate.ExecuteStatement(req);
      }
      else {
       ...
      }
  }
```


# Understanding communication with HS2 server

As mentioned in the [Thrift post](pauldeschacht.github.io/thrift/2014/02/27/Understanding-Trift.html#additional_information), the service binds a TTransport and a TProcessor together. The type of transport (TServerSocker, TSSLTransport, ...) is determined by the hive-site parameters file and is handled by the [HiveAuthFactory](https://github.com/pauldeschacht/hive/blob/cdh4.5.0-release/service/src/java/org/apache/hive/service/auth/HiveAuthFactory.java#L118).

The possible options are NONE, SASL, LDAP, KERBEROS and CUSTOM. As the multi tenant server depends on an internal authentication server, it must use the CUSTOM option. According the [documentation](http://www.cloudera.com/content/cloudera-content/cloudera-docs/CDH4/latest/CDH4-Security-Guide/cdh4sg_topic_9_1.html?scroll=concept_kzj_kc5_fm_unique_1), implementing the custom option requires writing a pluggable authentication
.
```Java
package org.apache.hive.service.auth;

import javax.security.sasl.AuthenticationException;

public interface PasswdAuthenticationProvider {
  /**
   * The Authenticate method is called by the HiveServer2 authentication layer
   * to authenticate users for their requests.
   * If a user is to be granted, return nothing/throw nothing.
   * When a user is to be disallowed, throw an appropriate {@link AuthenticationException}.
   *
   * For an example implementation, see {@link LdapAuthenticationProviderImpl}.
   *
   * @param user - The username received over the connection request
   * @param password - The password received over the connection request
   * @throws AuthenticationException - When a user is found to be
   * invalid by the implementation
   */
  void Authenticate(String user, String password) throws AuthenticationException;
}
```





The SASL layer takes care of the appropriate handshake according to the protocol (PLAIN, ANONYMOUS,...) and will use callback functions to get the user/password (client side) or to verify the user/password (server side)
