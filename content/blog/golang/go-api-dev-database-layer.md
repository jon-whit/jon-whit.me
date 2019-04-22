---
title: "Go API Development: Part 2 - The Database Layer"
date: 2019-04-21
categories: ["golang", "web", "api", "database", "sql"]
---

## Overview
Recall from the previous article that we are developing an API to manage Contacts. In [Go API Development: Part 1](../go-api-dev-routes/) we created a main application entry point that binds various API routes to HTTP handlers defined by an HTTP controller, and we wired up the controller with the interfaces it needs to do it's job. We also implemented a service struct that implements the [`ContactsManager`](https://github.com/jon-whit/go-contacts/blob/d05fa0ae711c663e761500ec313701357b262680/interfaces/ContactsManager.go#L5) interface.

In this article we will implement the [`ContactsDataAccessor`](https://github.com/jon-whit/go-contacts/blob/d05fa0ae711c663e761500ec313701357b262680/interfaces/ContactsDataAccessor.go#L5) interface so that we can start to actually persist the Contacts that we manage. We will implement this interface using [sqlite](https://www.sqlite.org/index.html) as our database backend.

<!--more-->

## Organizing Database Access
I stumbled upon a great [blog post](https://www.alexedwards.net/blog/organising-database-access) a while back on organizing database access. One of the suggestions was to create a Go interface for the database layer.

> The advantages of this [database interface] are twofold: first it gives our code a really clean structure, but – more importantly – it also opens up the potential to mock our database for unit testing.

Additionally, by creating an interface to the database layer you can swap out the database backend with any implementation that implements the interface without having to change any application code.

---

In the previous article we defined the [`ContactsDataAccessor`](https://github.com/jon-whit/go-contacts/blob/d05fa0ae711c663e761500ec313701357b262680/interfaces/ContactsDataAccessor.go#L5) interface.

```go
package interfaces
...

type ContactsDataAccessor interface {
	ListUserContacts(userID string) ([]models.Contact, error)
	CreateUserContact(userID string, contact models.Contact) (models.Contact, error)
}
```

This interface defines the database interface the [`ContactsService`](https://github.com/jon-whit/go-contacts/blob/d05fa0ae711c663e761500ec313701357b262680/services/ContactsService.go#L8) needs to persist the Contacts that it manages.

The last remaining bit for the database layer is that we need to implement the business logic needed to persist Contacts. To do that we'll break the `ContactsDataAccessor` implementation into two parts - the datastore layer and the database handler.

### ContactsDatastore
The [`ContactsDatastore`](https://github.com/jon-whit/go-contacts/blob/d05fa0ae711c663e761500ec313701357b262680/datastores/ContactsDatastore.go#L9) takes a [`DBHandler`](https://github.com/jon-whit/go-contacts/blob/da315963a9671119c5b4ffffe3effbf8b9464caa/interfaces/DBHandler.go#L5) interface. The idea here is that the datastore implementation shouldn't care what database backend is being used so long as it adheres to the [`DBHandler`](https://github.com/jon-whit/go-contacts/blob/da315963a9671119c5b4ffffe3effbf8b9464caa/interfaces/DBHandler.go#L5) interface.

[`datastores/ContactsDatastore.go`](https://github.com/jon-whit/go-contacts/blob/master/datastores/ContactsDatastore.go) - implements the business logic needed to persist Contacts.

```go
package datastores
...

type ContactsDatastore struct {
	interfaces.DBHandler
}

func (ds *ContactsDatastore) ListUserContacts(userID string) ([]models.Contact, error) {
	rows, err := ds.Query("SELECT (id, first_name, last_name, phone, email) FROM contacts WHERE user_id=?", userID)
	if err != nil {
		return nil, err
	}

	var contacts []models.Contact
	for rows.Next() {

		var id, firstName, lastName, phone, email string

		err = rows.Scan(&id, &firstName, &lastName, &phone, &email)
		if err != nil {
			return nil, err
		}

		contacts = append(contacts, models.Contact{
			ID:        id,
			FirstName: firstName,
			LastName:  lastName,
			Phone:     phone,
			Email:     email,
		})
	}

	return contacts, nil
}

func (ds *ContactsDatastore) CreateUserContact(userID string, contact models.Contact) (models.Contact, error) {
	_, err := ds.Execute(`INSERT INTO contacts (user_id, id, first_name, last_name, phone, email) VALUES (?,?,?,?,?,?)`,
		userID, contact.ID, contact.FirstName, contact.LastName, contact.Phone, contact.Email)

	if err != nil {
		return models.Contact{}, err
	}

	return contact, nil
}
```

### SQLiteHandler

Now the magical part! We implement the [`DBHandler`](https://github.com/jon-whit/go-contacts/blob/da315963a9671119c5b4ffffe3effbf8b9464caa/interfaces/DBHandler.go#L5) interface for sqlite.

[`datastores/SQLiteHandler.go`](https://github.com/jon-whit/go-contacts/blob/master/datastores/SQLiteHandler.go) - implements the [`DBHandler`](https://github.com/jon-whit/go-contacts/blob/da315963a9671119c5b4ffffe3effbf8b9464caa/interfaces/DBHandler.go#L5) interface for sqlite.
```go
package datastores
...

type SQLiteHandler struct {
	Conn *sql.DB
}

func (handler *SQLiteHandler) Execute(query string, args ...interface{}) (sql.Result, error) {
	return handler.Conn.Exec(query, args)
}

func (handler *SQLiteHandler) Query(query string, args ...interface{}) (interfaces.DBRow, error) {
	rows, err := handler.Conn.Query(query, args)

	if err != nil {
		return new(SqliteRow), err
	}
	row := new(SqliteRow)
	row.Rows = rows

	return row, nil
}

type SqliteRow struct {
	Rows *sql.Rows
}

func (r SqliteRow) Scan(dest ...interface{}) error {
	err := r.Rows.Scan(dest...)
	if err != nil {
		return err
	}

	return nil
}

func (r SqliteRow) Next() bool {
	return r.Rows.Next()
}
```

Tomorrow I may choose to swap over to PostgreSQL or MySQL instead.. All I'd have to do to support one of these databases is implement the `DBHandler` interface and the datastore layer would be all ready to go!

## Wiring Up the Database Layer
Now all we have to do is wire up the `SQLiteHandler`, `ContactsDatastore`, and the `ContactsService`. We do this in the `InitRouter` function defined in `router.go`.

```go
func (router *router) InitRouter() *chi.Mux {
    ...

    // Create the SQLite DB Handler
    sqlConn, err := sql.Open("sqlite3", "/var/tmp/go-contacts.db")
    if err != nil {
        // handle error
    }
    sqliteHandler := &datastores.SQLiteHandler{
        Conn: sqlConn,
    }

    // Inject all implementations of the interfaces.
    controller := controllers.ContactsController{
        &services.ContactsService{
            DataAccessor: &datastores.ContactsDatastore{
                sqliteHandler,
            },
        },
    }

    ...
```

---

## Summary
Now that we have implemented the database layer and wired it up with the `ContactService` implementation, we should have a fully functional Contacts API. We should be able to list a user's Contacts and create a Contact, and these changes are now persisted using a sqlite database. The only thing remaining is testing..

In ["Go API Development: Part 3"](../go-api-dev-testing.md) I'll demonstrate how we can leverage [golang/gomock](https://github.com/golang/mock) to generate mock implementations of the various interfaces we've authored up to this point. We'll use these interface mocks to inject dependencies for testing.

Don't be afraid to [check out the code](https://github.com/jon-whit/go-contacts) and review what we've done so far.
