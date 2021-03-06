pg_plsql_graphs
===============

_pg_plsql_graphs_ is an extension for _PostgreSQL_.

Its main purpuse is to:

- Generate **flow** and **dependence graphs** for **plpgsql functions**
- Make these graphs accessable to the user via a database view
- Make it possible to export these graphs to **dot** files that can be converted e.g. to **png** or **pdf** files


##Example 1
The following shows a snippet of a simple **plpgsql function**

```Sql
...
BEGIN
	a := b + c;
	if not a > 10 then
		d := b * e;
		e := d + 1;
	end if;
	d := e / 2;
	return d;	
END;
...
```

This function would generate the following **flow** and **dependence graphs** (here in **png** format)

#### Flow Graph
![Flow Graph](https://raw.githubusercontent.com/BA-KLI/pg_plsql_graphs/master/examples/ex1/flow.png)


#### Dependence Graph
![Dep. Graph](https://raw.githubusercontent.com/BA-KLI/pg_plsql_graphs/master/examples/ex1/dep.png)


##Example 2

The following shows a more complex **plpgsql function**


```Sql
CREATE OR REPLACE FUNCTION doTest2() returns float AS $$
DECLARE
	prices decimal[];
	price decimal;
	pk int[];
	size int;
	overallprice decimal := 0;
BEGIN
	FOR i in 1..40 LOOP
		pk := array_append(pk, i);
	END LOOP;
	
	FOREACH size IN array pk LOOP
		SELECT 	sum(P.p_retailprice) 
		INTO price FROM 	PART P
	  	WHERE 	P.p_size = size;
	  
	  	overallprice := overallprice + price;
	END LOOP;
	  
	  raise notice '%',overallprice;
	  
	RETURN 0;
END;
$$ LANGUAGE plpgsql;
```

This simple function would generate the following **flow** and **dependence graphs** (here in **png** format)

#### Flow Graph
![Flow Graph](https://raw.githubusercontent.com/BA-KLI/pg_plsql_graphs/master/examples/ex2/flow.png)


#### Dependence Graph
![Dep. Graph](https://raw.githubusercontent.com/BA-KLI/pg_plsql_graphs/master/examples/ex2/pdg.png)


##Configuration and Installation

- Download the _igraph_ library and install it to your local system. You can use the following link:
http://igraph.org/nightly/get/c/igraph-0.7.1.tar.gz

- Download the _PostgresSQL_ sources e.g. via **git** from [git://git.postgresql.org/git/postgresql.git](git://git.postgresql.org/git/postgresql.git) (More details are given here: https://wiki.postgresql.org/wiki/Working_with_Git)


- Add the configurations given by the following **diff** to **configure.in** of your _PostgresSQL_ sourcecode at the correct places. This will make the _igraph_ library available for _PostgresSQL_

```Diff
--- configure.in
+++ configure.in
@@ -727,6 +727,11 @@ fi
 AC_SUBST(with_uuid)
 AC_SUBST(UUID_EXTRA_OBJS)
 
+#
+# iGraph library
+#
+PGAC_ARG_BOOL(with, igraph, no, [build contrib/pg_plsql_graphs, requires IGraph library])
+AC_SUBST(with_igraph)
 
 #
 # XML
@@ -997,6 +1002,10 @@ elif test "$with_uuid" = ossp ; then
 fi
 AC_SUBST(UUID_LIBS)
 
+#for contrib/pg_plsql_graphs
+if test "$with_igraph" = yes; then
+	AC_CHECK_LIB([igraph], [igraph_adjlist_init], [], [echo "Library Igraph not found!"; exit -1])
+fi
 
 ##
 ## Header files
@@ -1142,6 +1151,13 @@ if test "$PORTNAME" = "win32" ; then
    AC_CHECK_HEADERS(crtdefs.h)
 fi
 
+
+# for contrib/pg_plsql_graphs
+if test "$with_igraph" = yes ; then
+    AC_CHECK_HEADERS(igraph/igraph.h, [],
+      [AC_MSG_ERROR([header file <igraph/igraph.h> is required for iGraph])])
+fi
+
 ##
 ## Types, structures, compiler characteristics
 ##

```

- Clone this git repository to the **contrib** folder. E.g. using the following command (assuming you are on the root folder of the _PostgreSQL_ source)

```Shell
cd ./contrib; git clone https://github.com/BA-KLI/pg_plsql_graphs.git
```

- Edit the Makefile in the **contrib** folder and add **pg_plsql_graphs** to **SUBDIRS**

```Shell
...
SUBDIRS = \
		pg_plsql_graphs	\
		adminpack	\
		auth_delay	\
		auto_explain	\
		btree_gin	\
...
```

- (Re)configure _PostgreSQL_ with **--with-igraph**, (re)make and (re)install it

- E.g. by by typing the following in the root folder of the _PostgreSQL_ source (replacing the *<INSTALLDIR>* variable with the directory where you want to install _PostgreSQL_):

```Shell
autoconf; ./configure --with-igraph   --prefix=<INSTALLDIR>; make; make install
```



- This should install _PostgreSQL_ for you. If you have problems or need more information take a look at http://www.postgresql.org/docs/9.3/static/installation.html


- Install the **contrib** extensions
```Shell
cd contrib; make; make install
```

- If you didn't already set up a the databasecluster and a database, do it using **initdb** and connect to it with  **psql** (http://www.postgresql.org/docs/9.3/interactive/app-initdb.html and http://www.postgresql.org/docs/9.3/interactive/sql-createdatabase.html)

- Locate your **postgres.conf** configuration file by starting up the **PostgreSQL Server** and then typing the following command in the **psql** client: 

```Shell
SHOW config_file;
```

- In the **postgres.conf** add **pg_plsql_graphs** to **shared_preload_libraries**. It should look like this:

```Shell
...
shared_preload_libraries = 'pg_plsql_graphs'   # (change requires restart)
...
```

- Restart the **PostgreSQL Server**



##Usage

- Start up **psql** and type 

```Sql
CREATE EXTENSION pg_plsql_graphs;
```

- Now for every **plpgsql function** you call, a corresponding entry with the **flow** and **depencence graphs** in **dot** format is created a HashTable that is accessable by the **pg_plsql_graphs** view.

- After calling a **plpgsql function** you can now query this view e.g. by typing: 

```Sql
SELECT * FROM pg_plsql_graphs;
```

- This will show you an indented version of the **dot graphs** of the previously called **plpgsql functions**. There is also a not indented version that is optimized for exports to external files called **pg_plsql_graphs_trimmed**.

- In order to export a graph to a external file (e.g. to convert it to png) you can use the COPY function. The following commands will copy the **flow** and **depence graph** of the last called **plpgsql function** to external **dot** files. To do that the views **pg_plsql_last_flowgraph_dot** and **pg_plsql_last_pdgs_dot** are used.

```Sql
\COPY (select * from pg_plsql_last_flowgraph_dot)  TO 'flow.dot';
\COPY (select * from pg_plsql_last_pdgs_dot)  TO 'pdg.dot';
```

- You can now convert these **dot** files e.g. to the **png** format using the following _graphviz_ commands

```Shell
dot -Tpng 'flow.dot' > flow.png
dot -Tpng 'pdg.dot' > pdg.png
```
