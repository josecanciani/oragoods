# OraGooDS
(Originally hosted in code.google.com/p/oragoods, automatically exported here)

## Summary

This project provides an Oracle PL/SQL package that developers can use to get <a href="http://code.google.com/apis/visualization/documentation/dev/implementing_data_source_overview.html">Google DataSources</a> using the <a href="http://code.google.com/apis/visualization/documentation/querylanguage.html">Google Query Language</a>, from any webpage directly from you database. It should be easy to modify it to serve other type of datasources (like YUI or similar).

With the help of the >=10g's Embedded Database Gateway (EPG) you can query directly to Oracle without any extra middle components (web servers) and get a DataSource in your application.

Even though the project is abandoned, it's fully funcional and feel free to submit bugs or push requests.

Here's a quick example of what you get (but keep reading, there's more ways to get the same thing):

<pre>

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

</pre>

## Installation

In order to use GDataSource you need to configure your Oracle Listener to answer HTTP calls and install the package.
 
* Configuring the DBMS_EPG (Embedded PL/SQL Gateway)

This code will create a new DAD (Database Access Descriptor) in your Oracle Instance. You need to run this as SYS (I guess any DBA user should work too). We will use user "SCOTT" for the test, but you will probably want to change the DAD's name and URL to something that suit your needs.

Note: watch out for the ? that the wiki adds automatically on the DebugStyle word! Remove it if you are copying and pasting.

<pre>

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
</pre>

This should automatically configure your listener (if you are using dynamic listener registration, which your DBA should know about). Usually the default port is 8080, so accesing your database server with your browser should work now: <b>http://your-db-server:8080/scott/home</b>

Here's a detailed note about DBMS_EPG if you have troubles: <a href="http://www.oracle-base.com/articles/10g/dbms_epg_10gR2.php">http://www.oracle-base.com/articles/10g/dbms_epg_10gR2.php</a>

* Download and Install GDataSource package

Follow instructions on how to checkout the code of this repo.

Then connect to your database and compile the packages:

<pre>

$ sqlplus /nolog

SQL> connect scott/tiger@YOURDB

SQL> @schema.sql

SQL> @gdatasource.pks

SQL> @gdatasource.pkb


</pre>

That's all! Now we can test it.

## Using the package

You can test it from SQL Plus or similar tool, and also from your web browser if the DBMS_EPG configuration was successful.

The schema.sql creates a table called gdatasources. It has just two columns: 1) datasource ID (a varchar2 string) and 2) sql_text where the datasource base sql query is defined.

The schema.sql already creates one datasource query for testing purpose, which we will use now.

<pre>

sqlplus /nolog

SQL> connect scott/tiger@YOURDB

SQL> set serveroutput on

SQL> begin
  2  gdatasource.g_debug := true;
  3  gdatasource.get_json('test','select `*`');
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

</pre>

As you can see, this returns a Google DataSource that you can use with the Visualization API for doing charts, printing tables, etc.

From the browser you can use it as: http://your-db-server:8080/scott/gdatasource.get_json?p_datasource_id=test

You can also get the output as a CLOB variable using PL/SQL. Here's an example, we print the resulting CLOB to screen with dbms_output, you probably will want to do something else:

<pre>
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
</pre>

And also as a pipelined function!

<pre>

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

</pre>

## Adding your own Data Sources

The schema.sql file creates a table (gdatasources). In order to use your own datasources you just need to insert new ones in this table. For example, for a list of employees names:

<pre>
insert into gdatasources values ('emp-names','select first_name, last_name from hr.employees');
commit;
grant select on hr.employees to scott;
</pre>

NOTE: OraGoods doesn't really care what you insert there, it can be any query Oracle supports (with inline views or whatever), but the outer select clause is parsed, so it must complain with these rules: 

1) if two or more columns have the same name, use the AS keyword to name them uniquely: select a.id AS myfirstid, b.id AS mysecondid ... ;

2) if your query contains functions then use an inline view to wrap them, ej: select col1, mysum2 from (select col1, sum(col2) AS mysum2 ... );

Don't forget to grant the proper privileges if the user your are using is not the owner of the tables.

Then just call the datasource by it's name: http://your-db-server:8080/your_dad/gdatasource.get_json?p_datasource_id=emp-names

You can also filter using the Google Query Language: http://your-db-server:8080/your_dad/gdatasource.get_json?p_datasource_id=emp-names&tq=select%20first_name%20where%20last_name%20%3D%20%22tiger%22

Here we encoded tq with the encodeURIComponent() javascript function. tq actually is: <b>select first_name where last_name = "tiger"</b>


Jose Canciani
http://www.4tm.biz
