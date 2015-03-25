# Introduction #

GDataSource is an Oracle PL/SQL implementation of a <a href='http://code.google.com/apis/visualization/documentation/dev/implementing_data_source_overview.html'>Google DataSource</a> "server". It implements the <a href='http://code.google.com/apis/visualization/documentation/querylanguage.html'>Google Query Language</a>.


# Details #

In order to use GDataSource you need to configure your Oracle Listener to answer HTTP calls and install the package.

## Configuring the DBMS\_EPG (Embedded PL/SQL Gateway) ##

This code will create a new DAD (Database Access Descriptor) in your Oracle Instance. You need to run this as SYS (I guess any DBA user should work too). We will use user "SCOTT" for the test, but you will probably want to change the DAD's name and URL to something that suit your needs.

Note: watch out for the ? that the wiki adds automatically on the DebugStyle word! Remove it if you are copying and pasting.

```


BEGIN

DBMS_EPG.create_dad (
dad_name => 'scott',
path     => '/scott/*');

DBMS_EPG.set_dad_attribute (
dad_name   => 'scott',
attr_name  => 'error-style',
attr_value => 'DebugStyle');

DBMS_EPG.set_dad_attribute (
dad_name   => 'scott',
attr_name  => 'authentication-mode',
attr_value => 'Basic');

DBMS_EPG.authorize_dad (
dad_name => 'scott',
user     => 'SCOTT');

END;

CREATE OR REPLACE PROCEDURE scott.home IS
BEGIN
HTP.htmlopen;
HTP.headopen;
HTP.title('This is a test page!');
HTP.headclose;
HTP.bodyopen;
HTP.print('This is a test page! DateTime: ' || TO_CHAR(SYSTIMESTAMP));
HTP.bodyclose;
HTP.htmlclose;
END home;
/


```

This should automatically configure your listener (if you are using dynamic listener registration, which your DBA should know about). Usually the default port is 8080, so accesing your database server with your browser should work now: <b><a href='http://your-db-server:8080/scott/home'>http://your-db-server:8080/scott/home</a></b>

Here's a detailed note about DBMS\_EPG if you have troubles: <a href='http://www.oracle-base.com/articles/10g/dbms_epg_10gR2.php'><a href='http://www.oracle-base.com/articles/10g/dbms_epg_10gR2.php'>http://www.oracle-base.com/articles/10g/dbms_epg_10gR2.php</a></a>

## Download and Install GDataSource package ##

Now you need to download the source code and install the GDataSource Package on the SCOTT schema.

You can download the latest stable version from the <a href='http://code.google.com/p/oragoods/downloads/list'>Downloads tab</a>, or you can try the lastest development snapshot. For the latter you'll need a svn command line client client. Just run this to get the source code:

```

svn checkout http://oragoods.googlecode.com/svn/trunk/ oragoods-read-only
```

Then connect to your database and compile the packages:

```


$ sqlplus /nolog

SQL> connect scott/tiger@YOURDB

SQL> @schema.sql

SQL> @gdatasource.pks

SQL> @gdatasource.pkb


```

That's all! Now we can test it.

## Using GDataSource ##

You can test it from SQL Plus or similar tool, and also from your web browser if the DBMS\_EPG configuration was successful.

The schema.sql creates a table called gdatasources. It has just two columns: 1) datasource ID (a varchar2 string) and 2) sql\_text where the datasource base sql query is defined.

The schema.sql already creates one datasource query for testing purpose, which we will use now.

```


sqlplus /nolog

SQL> connect scott/tiger@YOURDB

SQL> set serveroutput on

SQL> begin
2  gdatasource.g_debug := true;
3  gdatasource.get_json('test','select *');
4  end;
5  /
google.visualization.Query.setResponse(
{
version: "0.7",
status: "ok",
reqId: 0,
table: {
cols: [
{id: "COL1", label: "COL1", type: "string"},
{id: "COL2", label: "COL2",
type: "string"},
{id: "COL3", label: "COL3", type: "string"}

],
rows: [
{c: [
{v: "this"},
{v: "is"},
{v: "test"}
]}
]
}
}
)

PL/SQL procedure successfully completed.

```

As you can see, this returns a Google DataSource that you can use with the Visualization API for doing charts, printing tables, etc.

From the browser you can use it as: http://your-db-server:8080/scott/gdatasource.get_json?p_datasource_id=test

As of version 0.6 you can get the output as a CLOB variable using PL/SQL. Here's an example, we print the resulting CLOB to screen with dbms\_output, you probably will want to do something else:

```

declare
l_offset number default 1;
begin
gdatasource.g_debug := false;
gdatasource.g_opt_clob_output := true;
gdatasource.get_json('test','select *');
loop
exit when l_offset > dbms_lob.getlength(gdatasource.g_clob_output);
dbms_output.put_line( dbms_lob.substr( gdatasource.g_clob_output, 255, l_offset ) );
l_offset := l_offset + 255;
end loop;
end;
```

And also as a pipelined function!

```


SQL> select * from table(gdatasource.get_json('test','select *'));

COLUMN_VALUE
============
google.visualization.Query.setResponse(
{
version: "0.7",
status: "ok",
reqId: 0,
table: {
cols: [
{id: "COL1", label: "COL1", type: "string"},
{id: "COL2", label: "COL2", type: "string"},
{id: "COL3", label: "COL3", type: "string"}
],
rows: [
{c: [
{v: "this"},
{v: "is"},
{v: "test"}
]}
]
}
}
)

```

This code is new and has not much testing so far, so you can load bugs in the Issues tab.

Regards,
Jose Canciani
www.4tm.biz