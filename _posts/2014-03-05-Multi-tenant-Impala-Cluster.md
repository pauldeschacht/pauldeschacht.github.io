---
layout: post
title:  "Multi tenant Impala Cluster"
date:   2014-03-05
categories: impala 
---

The goal is to create a **multi tenant Impala cluster**. In other words, we need a server that authorizes each SQL statement for each user. Probably, [Cloudera Sentry](http://www.cloudera.com/content/cloudera/en/products-and-services/cdh/sentry.html) can play this role and might replace this server. 

The [multi tenant server](http://github.com/pauldeschacht/cloudera-hive) is not directly based on the Impala server (C++) but on Cloudera's HiveServer2 (Java). Both Impala server and HiveServer2 implement the same [TCLIService protocol](https://github.com/cloudera/Impala/blob/master/common/thrift/ImpalaService.thrift). This protocol is used in Cloudera's ODBC 2.5 driver. It was easier to modify the HiveServer2 code that the Impala code, therefore I took the [HiveServer2 implementation](https://github.com/cloudera/hive/tree/cdh4.5.0-release) as starting point.

The multi tenant server sits between the external ODBC clients and the actual Impala cluster. It inspects each SQL statement. The authorization itself is done by a remote server. If this server grants acces, the intermediate server will simply forward the request to the actual Impala cluster.

![Infrastructure multi tenant Impala cluster](/images/infra_impala.gif)

The multi tenant server is nothing more than a [delegator](http://en.wikipedia.org/wiki/Delegation_pattern):
1. Capture each SQL statement and verify with the authorization service.
2. If the access is authorized, forward the query to the Impala cluster, otherwise return an error.

## Impala server verser HiveServer2

The [Cloudera Impala](https://github.com/cloudera/Impala) server launches [2](https://github.com/cloudera/Impala/blob/master/be/src/service/impala-server.cc#L1773) external facing Thrift services

* Impala/TCLIService Thrift service (aka Hive Server 2 protocol)  on port 21050, used by ODBC 2.5
* Beeswax Thrift service (aka Hive Server protocol) on port 21000, used by ODBC 1.2

The [Cloudera HiveServer2](https://github.com/pauldeschacht/hive/tree/cdh4.5.0-release/service/src/java/org/apache/hive/service) launches one service

* TCLIService Thrift service on port 10000

In order to create a multi tenant server, it is sufficient to implement the TCLIService. There options are:

* Start from scratch and implement the Thrift TCLIService services. 
* Derive from Cloudera Impala server (however building Impala is not for the faint of heart)
* Derive from Cloudera HiveServer2

Building on top of Cloudera's HiveServer2 turned out to be fastest solution to get to a working prototype.

# Delegation of the Thrift commands

The idea is to delegate all the Thrift request (defined in the ImpalaService.thrift), except for 
* OpenSession
* CloseSession
* ExecuteStatement

These 3 functions will be intercepted and a specific action will be done before or after the delegation.

## Construction of the delegate

During initialization of the server, the delegate is constructed. The delegate will forward the requests to the actual Impala server.


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

The OpenSession request is first forwarded to the actual Impala server. The reply contains a session identifier which is then linked to the session's user information. A simple mapping from a session identifier to the user information is maintained.

```Java
  @Override
  public TOpenSessionResp OpenSession(TOpenSessionReq req) throws TException {
      TOpenSessionResp resp = delegate.OpenSession(req);
      // store the identifier of the Impala session
      String username = req.getUsername();
      String password = req.getPassword();
      ...
      sessions.addSession(username,password,ip,resp.getSessionHandle());
      return resp;
```

## Capture the SQL statement 

Each request that executes a SQL statement is intercepted. The mapping from session to user information tells which user executes the SQL statement. The remote authentication server informs us whether the user has the permission to execute the statement. If granted, the request is forwarded to the actual Impala server. Otherwise, a response with an appropriate message is constructed.

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
      TStatus status = TStatus(TStatusCode.ERROR_STATUS);
      status.setErrorMessage("No permission to execute statement");
      ...
      TExecuteStatementResp response = new TExecuteStatementResp(status);
      response.setStatus(status);
      return response;
      }
  }
```


# A small note on the secure communication with HiveServer2 server

As mentioned in the [Thrift post](pauldeschacht.github.io/thrift/2014/02/27/Understanding-Trift.html#additional_information), the server binds a TTransport and a TProcessor together. The type of transport (TServerSocker, TSSLTransport, ...) is determined by the hive-site parameters file and is handled by the [HiveAuthFactory](https://github.com/pauldeschacht/hive/blob/cdh4.5.0-release/service/src/java/org/apache/hive/service/auth/HiveAuthFactory.java#L118).

## SSL 

SSL is handled by the Thrift transport. It does the initial handshake before accepting any communication. HiveServer2 supports the [following options](https://github.com/pauldeschacht/hive/blob/cdh4-0.10.0_4.5.0/service/src/java/org/apache/hive/service/auth/HiveAuthFactory.java#L50) in hive-site.xml

## HiveServer2 authentication

* "NOSASL": no authentication at all, uses the default TTransport transport
* "NONE": SASL PLAIN  (default), uses the PlainTransport that implements the SASL handshake (using callbacks)
* "LDAP": SASL PLAIN
* "CUSTOM": SASL PLAIN
* "Kerberos": SASL GSSAPI

## Impala Server authentication 

As described in the [documentation](http://www.cloudera.com/content/cloudera-content/cloudera-docs/Connectors/PDF/Cloudera-ODBC-Driver-for-Impala-Install-Guide.pdf), Impala server supports the following authentication methods
* No Authentication
* User Name (it merely labels a session)
* Kerberos

# Two use cases

## Client with authentication

If the client can authenticate, it is enough the deploy a single multi tenant server. The generic ODBC 2.5 driver supports 

* username
* username/password
* username/password/SSL
* Kerberos

![](/images/Multitenant_Security_Thrift.gif)

## Client without authentication

In case the client cannot authenticate (for example Tableau Impala ODBC driver does not allow to authenicate over SSL), a multi tenant server is deployed for each client. Each client targets a dedicated port and a dedicated multi tenant server. Each multi tenant server can use the port (or an alias) to authorize the query.

![](/images/Multitenant_Security_SSL.gif)

Of course, it is possible to find intermediate solutions, such as Impala ODBC with user name authentication over a SSL tunnel.
