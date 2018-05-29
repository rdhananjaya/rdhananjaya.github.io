---
layout: post
title:  "JDBC Driver: Introduction"
summary: "Introduction to JDBC driver internals, will try to expand this on later posts."
date:   2018-04-05 19:16:36 +0530
categories: design
comments: true
page_id: jdbc
---

# Introduction to JDBC drivers.

JDBC (Java Database Connectivity) is the standard Java abstraction for database connectivity from java programs. Almost all popular relational database vendors provide a JDBC driver to connect to their database. JDBC drivers are nothing but implementation of interfaces defined in ‘java.sql’ package. Driver, Connection, Statement, ResultSet and few more.

JDBC drivers are categorized into 4 types:

- Type 1: JDBC-ODBC (Open Database Connectivity) Bridge
- Type 2: Native-API, partly Java driver
- Type 3: Network-protocol, all-Java driver (connects to a middle-tier)
- Type 4: Native-protocol, all-Java driver (connects to a database)

> Side Note: If you wonder what ODBC is it’s a standard API developed by Microsoft and a company called Simba Technologies during early 90s, which allow client applications (in Two-Tier architecture) to directly connect to the database. ODBC usually employee device driver like mechanism to expose functionality to the user land applications. Where we as users (developers) get some statically or dynamically linked library that exposes the API, which in turn talks to the database via ODBC driver, which act as a device driver. (https://en.wikipedia.org/wiki/Open_Database_Connectivity)

Most of the widely-used JDBC drivers tend to be of Type-4, the JDBC driver communicates with the database using database’s native protocol and the driver is implemented in pure Java.
Do you remember when you want to use the JDBC driver from JavaSE you first have to register the driver by loading the driver class `Class.forName("com.mysql.jdbc.Driver")`. Class.forName static method will load the given class into JVM, by doing so static block inside ‘Driver’ class is executed as per java class loading rules. This static block contains code to register our JDBC driver in ‘DriverManager’ class.
When we do `Connection dummyCon = DriverManager.getConnection(dbUrl)` the DriverManger class make sure to give us an instance of Connection of requested driver. (it works like a stringify factory method). Under the hoods DriverManager ask each of registered Drivers weather it would accept given DbUrl by invoking Driver.connect(dbUrl) method and when it found an accepting Driver, the DriverManager will return that Connection instance.

Let’s start our investigation with Sqlite driver.
You should be able to find the source code here https://github.com/xerial/sqlite-jdbc
