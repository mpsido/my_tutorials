Create a database

    mkdir database
    pg_ctl init -D$PWD/database
    pg_ctl start -D$PWD/database
    createdb $USER # if not You may need to add "-d postgress" to the psql commands

Database console
    
    psql <database name> (by default it may be postgres)

Database [graphic client](https://wiki.postgresql.org/wiki/Apt)
    
    pgadmin3


#Insert data into db:

    INSERT INTO marketdata.cryptowatch(entry_id, price, volume, entry_time) VALUES (?, ?, ?, ?);

#Python script to read/write database

    import psycopg2

    try:
        connect_str = "dbname='postgres' user='mbenthaier' host='localhost' "
        # use our connection values to establish a connection
        conn = psycopg2.connect(connect_str)
        # create a psycopg2 cursor that can execute queries
        cursor = conn.cursor()
        # create a new table with a single column called "name"
        cursor.execute("""INSERT INTO marketdata.cryptowatch(entry_id, price, volume, entry_time) VALUES (1, 6, 41, '2018-03-01');""")
        # run a SELECT statement - no data in there, but we can try it
        cursor.execute("""SELECT * from marketdata.cryptowatch""")
        rows = cursor.fetchall()
        print(rows)

        conn.commit() #if you forget this the changes to the database may not be persistent
    except Exception as e:
        print("Uh oh, can't connect. Invalid dbname, user or password?")
        print(e)


# Start postgraphile server 

    postgraphile -c $USER:localhost:5432/<database name> -s <schema name>

On the browser try url: localhost:5000/graphiql (should show a webapp)


### Manually query postgraphile server

    alias ql='curl -X POST -H "Content-Type: application/graphql" -d'
    ql '{ allEntries { edges { node { projectName, resource } }}}' http://localhost:5000/graphql

Query with filter.
    ql '{ allEntries(condition: {resource: "Ricky"}) { edges { node {projectName, resource }}}}' http://localhost:5000/graphql
