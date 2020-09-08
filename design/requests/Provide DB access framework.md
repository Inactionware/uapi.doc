## Proposal

Database access is basic function in application, normally we access database in Java code is JDBC or Hibernate ORM library.  
JDBC is native SQL based library, we have to optimize SQL for different database.  
Hiberante is OR mapping tool, but it become more complex today and you have to understand how it configures to provide better performance.  
So the idea is provide a simple DB access framework in UAPI, it is able to:

*   SQL based library, no OR mapping
*   User can add customized SQL easily
*   It is easy to extend to support multiple database

## Solution

TODO

## Reference

*   Cool JDBC library: [https://brettwooldridge.github.io/HikariCP/](https://brettwooldridge.github.io/HikariCP/)
*   Android ORM library: [http://satyan.github.io/sugar/index.html](http://satyan.github.io/sugar/index.html)