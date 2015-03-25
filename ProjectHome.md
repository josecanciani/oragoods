This project provides an Oracle PL/SQL package that developers can use to get Google Datasources using the Google Query Language.


With the help of the 10g's Embedded Database Gateway (EPG) you can query directly to Oracle without any extra middle components (web servers) and get a DataSource in your application.


Go to the Wiki for details on how to install and use it.

Here's a quick example of what you get (but checkout the wiki, there's more ways to get the same thing):

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
