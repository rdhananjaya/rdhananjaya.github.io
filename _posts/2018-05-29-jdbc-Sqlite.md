---
layout: post
title:  "JDBC Driver: Exploring SQLite driver"
summary: "Looking in the SQLite JDBC driver."
date:   2018-05-29 16:41:36 +0530
categories: design
comments: true
page_id: jdbc
---


We start from the `Driver` class. org.sqlite.JDBC

``` java
public class JDBC implements Driver
{
    public static final String PREFIX = "jdbc:sqlite:";

    static {
        try {
            DriverManager.registerDriver(new JDBC());
        }
        catch (SQLException e) {
            e.printStackTrace();
        }
    }
```

Just like any other driver it register it self in the `DriverManager`.
isValid method is set to check whether the given URL has a prefix of `PREFIX`.
createConnection will just instantiate `SQLiteConnection` object providing the URL and the properties bag usually contains other information like Username/Password.

``` java
public static Connection createConnection(String url, Properties prop) throws SQLException {
    if (!isValidURL(url))
        return null;

    url = url.trim();
    return new SQLiteConnection(url, extractAddress(url), prop);
}
```

### org.sqlite.SQLiteConnection 

``` java
import org.sqlite.jdbc4.JDBC4Connection;

public class SQLiteConnection extends JDBC4Connection
{
    /**
     * Constructor to create a connection to a database at the given location.
     * @param url The location of the database.
     * @param fileName The database.
     * @throws SQLException
     */
    public SQLiteConnection(String url, String fileName) throws SQLException {
        this(url, fileName, new Properties());
    }

    /**
     * Constructor to create a pre-configured connection to a database at the
     * given location.
     * @param url The location of the database file.
     * @param fileName The database.
     * @param prop The configurations to apply.
     * @throws SQLException
     */
    public SQLiteConnection(String url, String fileName, Properties prop) throws SQLException {
        super(url, fileName, prop);
    }
    ....
```

SQLiteConnection class just pass down those information into its base class, let's see what is JDBC4Connection class looks like.

``` java
package org.sqlite.jdbc4;

import java.sql.Array;
import java.sql.Blob;
import java.sql.Clob;
import java.sql.Connection;
import java.sql.DatabaseMetaData;
import java.sql.PreparedStatement;
import java.sql.SQLException;

import java.sql.Statement;
import java.util.Properties;

import org.sqlite.SQLiteConnection;
import org.sqlite.jdbc3.JDBC3Connection;

public abstract class JDBC4Connection extends JDBC3Connection implements Connection {

    public JDBC4Connection(String url, String fileName, Properties prop) throws SQLException
    {
        super(url, fileName, prop);
    }

    public DatabaseMetaData getMetaData() throws SQLException {
        checkOpen();

        if (meta == null) {
            meta = new JDBC4DatabaseMetaData((SQLiteConnection)this);
        }

        return (DatabaseMetaData)meta;
    }

    public Statement createStatement(int rst, int rsc, int rsh) throws SQLException {
        checkOpen();
        checkCursor(rst, rsc, rsh);

        return new JDBC4Statement((SQLiteConnection)this);
    }

    public PreparedStatement prepareStatement(String sql, int rst, int rsc, int rsh) throws SQLException {
        checkOpen();
        checkCursor(rst, rsc, rsh);

        return new JDBC4PreparedStatement((SQLiteConnection)this, sql);
    }
    ...
```

Ah ha, more delegations!! and we can see that JDBC4Connection implements `java.sql.Connection` interface and if you look inside the interface it's a classic example of
an abstract factory pattern. Anything that implements `Connection` interface can be treated as a JDBC connection (we do need to consider the semantics and correctness).

``` java
package java.sql;

public interface Connection extends java.sql.Wrapper, java.lang.AutoCloseable {
    int TRANSACTION_NONE = 0;
    int TRANSACTION_READ_UNCOMMITTED = 1;
    int TRANSACTION_READ_COMMITTED = 2;
    int TRANSACTION_REPEATABLE_READ = 4;
    int TRANSACTION_SERIALIZABLE = 8;

    java.sql.Statement createStatement() throws java.sql.SQLException;

    java.sql.PreparedStatement prepareStatement(java.lang.String s) throws java.sql.SQLException;

    java.sql.CallableStatement prepareCall(java.lang.String s) throws java.sql.SQLException;

    java.lang.String nativeSQL(java.lang.String s) throws java.sql.SQLException;

    void setAutoCommit(boolean b) throws java.sql.SQLException;

    boolean getAutoCommit() throws java.sql.SQLException;

    void commit() throws java.sql.SQLException;

    void rollback() throws java.sql.SQLException;

    void close() throws java.sql.SQLException;

    boolean isClosed() throws java.sql.SQLException;
    ...
```

Most prominent methods from this interface are `createStatement`, `prepareStatement`, `prepareCall`, and some overloads of the same methods.

### Now from here let's dive into JDBC3Connection class.

Any JDBC4Connection is a JDBC3Connection and most of the logic seems to be coming from the base class.
``` java
public abstract class JDBC3Connection extends CoreConnection {

	private final AtomicInteger savePoint = new AtomicInteger(0);

    protected JDBC3Connection(String url, String fileName, Properties prop) throws SQLException {
        super(url, fileName, prop);
    }

    public int getTransactionIsolation() {
        return transactionIsolation;
    }

    public void setTransactionIsolation(int level) throws SQLException {
        checkOpen();

        switch (level) {
        case java.sql.Connection.TRANSACTION_SERIALIZABLE:
            db.exec("PRAGMA read_uncommitted = false;");
            break;
        case java.sql.Connection.TRANSACTION_READ_UNCOMMITTED:
            db.exec("PRAGMA read_uncommitted = true;");
            break;
        default:
            throw new SQLException("SQLite supports only TRANSACTION_SERIALIZABLE and TRANSACTION_READ_UNCOMMITTED.");
        }
        transactionIsolation = level;
    }

    public abstract DatabaseMetaData getMetaData() throws SQLException;

    public String nativeSQL(String sql) {
        return sql;
    }

    public void clearWarnings() throws SQLException {}

    public SQLWarning getWarnings() throws SQLException {
        return null;
    }

    public boolean getAutoCommit() throws SQLException {
        checkOpen();
        return autoCommit;
    }

    public void setAutoCommit(boolean ac) throws SQLException {
        checkOpen();
        if (autoCommit == ac)
            return;
        autoCommit = ac;
        db.exec(autoCommit ? "commit;" : beginCommandMap.get(transactionMode));
    }

    public void commit() throws SQLException {
        checkOpen();
        if (autoCommit)
            throw new SQLException("database in auto-commit mode");
        db.exec("commit;");
        db.exec(beginCommandMap.get(transactionMode));
    }

    public void rollback() throws SQLException {
        checkOpen();
        if (autoCommit)
            throw new SQLException("database in auto-commit mode");
        db.exec("rollback;");
        db.exec(beginCommandMap.get(transactionMode));
    }

    public Statement createStatement() throws SQLException {
        return createStatement(ResultSet.TYPE_FORWARD_ONLY, ResultSet.CONCUR_READ_ONLY,
                ResultSet.CLOSE_CURSORS_AT_COMMIT);
    }

    public Statement createStatement(int rsType, int rsConcurr) throws SQLException {
        return createStatement(rsType, rsConcurr, ResultSet.CLOSE_CURSORS_AT_COMMIT);
    }

    public abstract Statement createStatement(int rst, int rsc, int rsh) throws SQLException;

    public CallableStatement prepareCall(String sql) throws SQLException {
        return prepareCall(sql, ResultSet.TYPE_FORWARD_ONLY, ResultSet.CONCUR_READ_ONLY,
                ResultSet.CLOSE_CURSORS_AT_COMMIT);
    }

    public CallableStatement prepareCall(String sql, int rst, int rsc, int rsh) throws SQLException {
        throw new SQLException("SQLite does not support Stored Procedures");
    }

    public PreparedStatement prepareStatement(String sql, int rst, int rsc) throws SQLException {
        return prepareStatement(sql, rst, rsc, ResultSet.CLOSE_CURSORS_AT_COMMIT);
    }

    public abstract PreparedStatement prepareStatement(String sql, int rst, int rsc, int rsh) throws SQLException;

    public Savepoint setSavepoint() throws SQLException {
    	checkOpen();
    	if (autoCommit)
    		// when a SAVEPOINT is the outermost savepoint and not
    		// with a BEGIN...COMMIT then the behavior is the same
    		// as BEGIN DEFERRED TRANSACTION
    		// http://www.sqlite.org/lang_savepoint.html
    		autoCommit = false;
    	Savepoint sp = new JDBC3Savepoint(savePoint.incrementAndGet());
    	db.exec(String.format("SAVEPOINT %s", sp.getSavepointName()));
    	return sp;
    }

    public Savepoint setSavepoint(String name) throws SQLException {
    	checkOpen();
    	if (autoCommit)
    		// when a SAVEPOINT is the outermost savepoint and not
    		// with a BEGIN...COMMIT then the behavior is the same
    		// as BEGIN DEFERRED TRANSACTION
    		// http://www.sqlite.org/lang_savepoint.html
    		autoCommit = false;
    	Savepoint sp = new JDBC3Savepoint(savePoint.incrementAndGet(), name);
    	db.exec(String.format("SAVEPOINT %s", sp.getSavepointName()));
    	return sp;
    }

    public void releaseSavepoint(Savepoint savepoint) throws SQLException {
    	checkOpen();
    	if (autoCommit)
    		throw new SQLException("database in auto-commit mode");
    	db.exec(String.format("RELEASE SAVEPOINT %s", savepoint.getSavepointName()));
    }

    public void rollback(Savepoint savepoint) throws SQLException {
    	checkOpen();
    	if (autoCommit)
    		throw new SQLException("database in auto-commit mode");
    	db.exec(String.format("ROLLBACK TO SAVEPOINT %s", savepoint.getSavepointName()));
    }
}
```
From the above code I removed bunch of methods for brevity.

It seems most of the communications are handled in `this` classes base class CoreConnection via `db` field.
This `db` (of type DB) seems like it's the SQLite database object that handle core DB logic or a proxy to it.

### Yup, let's look at CoreConnection

``` java
package org.sqlite.core;

import java.io.File;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.net.MalformedURLException;
import java.net.URISyntaxException;
import java.net.URL;
import java.sql.Connection;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.EnumMap;
import java.util.Map;
import java.util.Properties;
import java.util.Set;
import java.util.TreeSet;

import org.sqlite.date.FastDateFormat;

import org.sqlite.SQLiteConfig;
import org.sqlite.SQLiteConfig.DateClass;
import org.sqlite.SQLiteConfig.DatePrecision;
import org.sqlite.SQLiteConfig.Pragma;
import org.sqlite.SQLiteConfig.TransactionMode;
import org.sqlite.SQLiteConnection;

public abstract class CoreConnection {
    private static final String RESOURCE_NAME_PREFIX = ":resource:";

    private final String url;
    private String fileName;
    protected DB db = null;
    protected CoreDatabaseMetaData meta = null;
    protected boolean autoCommit = true;
    protected int transactionIsolation = Connection.TRANSACTION_SERIALIZABLE;
    private int busyTimeout = 0;
    protected final int openModeFlags;
    protected TransactionMode transactionMode = TransactionMode.DEFFERED;

    protected final static Map<TransactionMode, String> beginCommandMap =
        new EnumMap<SQLiteConfig.TransactionMode, String>(SQLiteConfig.TransactionMode.class);

    private final static Set<String> pragmaSet = new TreeSet<String>();
    static {
        beginCommandMap.put(TransactionMode.DEFFERED, "begin;");
        beginCommandMap.put(TransactionMode.IMMEDIATE, "begin immediate;");
        beginCommandMap.put(TransactionMode.EXCLUSIVE, "begin exclusive;");

        for (Pragma pragma : Pragma.values()) {
        	pragmaSet.add(pragma.pragmaName);
        }
    }

    /* Date storage configuration */
    public final DateClass dateClass;
    public final DatePrecision datePrecision; //Calendar.SECOND or Calendar.MILLISECOND
    public final long dateMultiplier;
    public final FastDateFormat dateFormat;
    public final String dateStringFormat;

    protected CoreConnection(String url, String fileName, Properties prop) throws SQLException
    {
        this.url = url;
        this.fileName = extractPragmasFromFilename(fileName, prop);

        SQLiteConfig config = new SQLiteConfig(prop);
        this.dateClass = config.dateClass;
        this.dateMultiplier = config.dateMultiplier;
        this.dateFormat = FastDateFormat.getInstance(config.dateStringFormat);
        this.dateStringFormat = config.dateStringFormat;
        this.datePrecision = config.datePrecision;
        this.transactionMode = config.getTransactionMode();
        this.openModeFlags = config.getOpenModeFlags();

        open(openModeFlags, config.busyTimeout);

        if (fileName.startsWith("file:") && !fileName.contains("cache="))
        {   // URI cache overrides flags
            db.shared_cache(config.isEnabledSharedCache());
        }
        db.enable_load_extension(config.isEnabledLoadExtension());

        // set pragmas
        config.apply((Connection)this);

    }
    ...
```

Looking at the import statements we can assume that this class perform some IO (both network and disk) and uses behavior/logic from other SQLite classes.
