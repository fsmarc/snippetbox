# Snippetbox
This is a demo web-application from the Book: ["Let's Go! - Learn build professional web applications with Go"](https://lets-go.alexedwards.net)
by _Alex Edwards_.

# Alterations
Otherwise in the book im using docker containers to run the application and its database. So some steps deviate from the book.

## Container setup and launching application
docker-compose is used to start the web-app and the mysql-database (from chapter 4) simultaneously.
Though ```docker-compose up``` will not launch the web-app. Therefore you must attatch to the web-apps container and start the app manually:
```
> docker attach snippetbox
# inside container
/go # cd src
# start app
/go/src # go run ./cmd/web -dsn ${DSN}
```
> The project folder is mountet to the _snippetbox_ container as a volume.
> So the container haven't be rebuild by src-code-changes on the host-machine.

## CH. 4.1 - Setting Up MySql
Because we are using the official MySql docker image, there is no need to install MySql on the host.
To scaffold the database attach to the database-container with ```docker attach snippetbox_db```and launch the MySql-client with
```mysql -u root -p```. The default root-password is "root", but you can change it in the docker-compose.yml file.

### Creating a New User
Unlike in the book, our database and web-app are not on the same host. So our web-app will not be able to connect to the database with 
the user ```'web'@'localhost'```. We can simply create a user that could connect from any host by replacing _localhost_ with _%_. Instead of
_pass_ we choose _web_ as our password:
```
CREATE USER 'web'@'%';
GRANT SELECT, INSERT, UPDATE ON snippetbox.* TO 'web'@'%';
-- Important: Make sure to swap 'pass' with a password of your own choosing.
ALTER USER 'web'@'%' IDENTIFIED BY 'web';
```

## CH. 4.3 - Creating a Database Connection Pool
Because our database is at another host and we createy a slightly different user as in the book, we must edit the data source name to
```web:web@(snippetbox_db)/snippetbox?parseTime=true```:
```
// The sql.Open() function initializes a new sql.DB object, which is essentially a
// pool of database connections.
db, err := sql.Open("mysql", "web:web@(snippetbox_db)/snippetbox?parseTime=true")
if err != nil {
    ...
}
```
The altered dsn is provided as an environment variable inside the snippetbox container. Therefore we can use the default dsn from the book and inject our dsn via dsn-flag at app-launching:
```
/go/src # go run ./cmd/web -dsn ${DSN}
```

## CH. 13.6 - Integration Testing

Like ub CH 4.1 there are some changes for user creation and the dsn.

### Creating test user
Instead of _localhost_ we use _%_ again:
```
CREATE USER 'test_web'@'%';
GRANT CREATE, DROP, ALTER, INDEX, SELECT, INSERT, UPDATE, DELETE ON test_snippetbox.* TO 'test_web'@'%';
ALTER USER 'test_web'@'%' IDENTIFIED BY 'web';
```
> NOTE: We choose web again for our password.

### Writing newTestDB function
Like for our production connection pool we must pass the host information in our
test connection pool:
 ```
 func newTestDB(t *testing.T) (*sql.DB, func()) {
    // Establish a sql.DB connection pool for our test database. Because our
    // setup and teardown scripts contains multiple SQL statements, we need
    // to use the `multiStatements=true` parameter in our DSN. This instructs
    // our MySQL database driver to support executing multiple SQL statements
    // in one db.Exec()` call.
    db, err := sql.Open("mysql", "test_web:web@(snippetbox_db)/test_snippetbox?parseTime=true&multiStatements=true")
    if err != nil {
        t.Fatal(err)
    }
    //...
 ```

 # Additional Notes
 The used golang-docker-image for our prototyping and testing is huge (810MB).
 The reason is, that this image has Go and all its dependencies installt. Therefore we can run our ```go run``` and ```go test``` commands inside a container.

 For a production container this is unnessecary overhead. A good idea may be (cross) compiling the go application on the host machine and pack it into a small alpine or even scratch image.