---
title: "Go API Development: Part 1 - Routes & Handlers"
date: 2019-02-09
categories: ["golang", "web", "api"]
---
## Overview
This is the first article in a series of articles on developing a web API using [Golang](https://golang.org/). In this series I will walk you through developing a simple REST API in Go using the Service Pattern with complete examples of Dependency Injection, Mocking, and a simple Redis Caching layer. We'll be using [sqlite](https://www.sqlite.org/index.html) as our database backend.

<!--more-->

If you wish to skip the articles and prefer to just read the code, you can find the [`go-contacts`](https://github.com/jon-whit/go-contacts) project on GitHub.

In this article I am going to give an overview of the project layout and also demonstrate how to setup the API routes and HTTP handlers for the API.

### Contacts API
We will be developing a REST API to manage Contacts. The Contacts API specification is defined as follows:


| Method | Path                                 | Description                    |
|--------|--------------------------------------|--------------------------------|
| GET    | `/users/:userid/contacts`            | List a user's contacts         |
| POST   | `/users/:userid/contacts`            | Create a contact               |

---------

## Project Layout
Before we jump right in, I want to give you an overview of how I like to organize my projects. This should help you understand some of the rationale behind why I put code where I do, and hopefully it will become apparent why certain code goes into certain files/packages.

I like to organize Go web API projects using this folder structure:

```bash
|- controllers
|- datastores
|- interfaces
|- models
|- mocks
|- services
main.go
router.go
```

This folder structure is inspired from [Ichsan Rahardianto's](https://github.com/irahardianto) [`service-pattern-go`](https://irahardianto.github.io/service-pattern-go/) page.

> The folder structure is created to accomodate seperation of concern principles, where every struct should have single responsibility to achieve decoupled system.

> Every folder is a namespace of their own, and every file / struct under the same folder should only use the same namepace as their root folder.

----

`main.go` - The main entry point of the Contacts API. The main method triggers the ChiRouter singleton and initializes (binds) the routes and handlers by calling InitRouter.
```go
package main
...

func main() {
	http.ListenAndServe(":8080", NewChiRouter().InitRouter())
}

```

`router.go` -  Binds the controller's handlers to the appropriate route to handle the HTTP request. In this example we're using the [Chi](https://github.com/go-chi/chi) router, which is a "lightweight, idiomatic and composable router for building Go HTTP services." I like using Chi because it's 100% compliant with the standard Go [net/http](https://golang.org/pkg/net/http/) library, which would allow me to swap it out for another net/http compliant library if I so choose.
```go
package main
...

type ChiRouter interface {
	InitRouter() *chi.Mux
}

type router struct{}

func (router *router) InitRouter() *chi.Mux {

	// Create the SQLite DB Handler
	// Covered in the next article in this series.
	var sqliteHandler interfaces.DBHandler

	// Inject all implementations of the interfaces.
	controller := controllers.ContactsController{
		&services.ContactsService{
			DataAccessor: &datastores.ContactsDatastore{
				sqliteHandler,
			},
		},
	}

	// Define and bind the API routes for the Contacts API
	r := chi.NewRouter()
	r.Get("/users/{userid}/contacts", controller.ListUserContacts)
	r.Post("/users/{userid}/contacts", controller.CreateContact)

	return r
}

var (
	m          *router
	routerOnce sync.Once
)

// NewChiRouter defines a Singleton, ensuring only a single ChiRouter is created
func NewChiRouter() ChiRouter {
	if m == nil {
		routerOnce.Do(func() {
			m = &router{}
		})
	}
	return m
}

```

## Controllers
Controllers are responsible for handling the HTTP requests coming into the router. The controller layer should not implement service logic and data access. The service and data access layers should be done seperately..

Controllers must implement services through their interface. Service interface implementations should NOT be done in the controller so as to maintain decoupled logic. The implementation will be injected during compile time.

`controllers/ContactsController.go`
```go
package controllers
...

type ContactsController struct {
	ContactManager interfaces.ContactsManager
}

func (controller *ContactsController) ListUserContacts(w http.ResponseWriter, req *http.Request) {

	userID := chi.URLParam(req, "userid")

	contacts, err := controller.ContactManager.ListUserContacts(userID)
	if err != nil {
		// Handle error
	}

	json.NewEncoder(w).Encode(contacts)
	return
}

func (controller *ContactsController) CreateContact(w http.ResponseWriter, req *http.Request) {

	userID := chi.URLParam(req, "userid")
	contact := models.Contact{}

	// See go-chi/render package
	if err := render.Bind(req, &contact); err != nil {
		// Handle error
	}

	createdContact, err := controller.ContactManager.CreateContact(userID, contact)
	if err != nil {
		// Handle error
	}

	json.NewEncoder(w).Encode(createdContact)
}
```

## Interfaces
`interfaces/ContactsManager.go` - Defines the interface to manage Contacts. The `ContactsService` struct will implement this interface.
```go
package interfaces
...

type ContactsManager interface {
	ListUserContacts(userID string) ([]models.Contact, error)
	CreateContact(userID string, contact models.Contact) (models.Contact, error)
}

```

`interfaces/ContactsDataAccessor.go` - Defines the data access layer interface to manage the persistence of Contact information. The `ContactsService` struct embeds this interface to interact with the data access layer.
```go
package interfaces
...

type ContactsDataAccessor interface {
	ListUserContacts(userID string) ([]models.Contact, error)
	CreateUserContact(userID string, contact models.Contact) (models.Contact, error)
}
```

`interfaces/DBHandler.go` - Defines the interfaces to interact with a SQL database backend. We will implement this interface in the next article in the series.
```go
package interfaces
...

type DBHandler interface {
    Query(query string, args ...interface{}) (DBRow, error)
    Execute(query string, args ...interface{}) (sql.Result, error)
}

type DBRow interface {
    Scan(dest ...interface{}) error
    Next() bool
}
```

## Models
The models folder houses the structs under the `models` namespace. Each model defines a struct that reflects the data object(s) serialized and deserialized to/from the database layer.

`models/users.go` - Defines the model for a User in the API.
```go
package models

type User struct {
	ID       string    `json:"id"`
	Contacts []Contact `json:"contacts"`
}
```

`models/contacts.go` - Defines the model for a Contact in the API.
```go
package models
...

type Contact struct {
	ID        string `json:"id"`
	FirstName string `json:"firstName"`
	LastName  string `json:"lastName"`
	Phone     string `json:"phone"`
	Email     string `json:"email,omitempty"`
}

// Bind on Contact will run after the unmarshalling is complete, its
// a good time to focus some post-processing after a decoding.
func (c *Contact) Bind(r *http.Request) error {
	return nil
}
```

## Services
The services folder houses structs under the `services` namespace. This folder is where the business logic should live. Structs living in this folder should handle the fetching of data from the data access layer and run the business logic needed to satisfy what the controller expects the Contacts API to return.

`services/ContactsService.go` - Defines the service which implements the `ContactManager` interface. In this example there isn't any business logic beyond interacting with the data access layer. However, in your application this may not be the case..
```go
package services
...

type ContactsService struct {
	DataAccessor interfaces.ContactsDataAccessor
}

func (service *ContactsService) ListUserContacts(userID string) ([]models.Contact, error) {
	return service.DataAccessor.ListUserContacts(userID)
}

func (service *ContactsService) CreateContact(userID string, contact models.Contact) (models.Contact, error) {
	return service.DataAccessor.CreateUserContact(userID, contact)
}

```

## Datastores
The datastores folder houses structs under `datastores` namespace. This folder is where the implementation of the data access layer should live. All queries and data operation to/from the database should happen here, and the implementation should be agnostic of what backend database engine is used and how the queries are done.

`datastores/ContactsDatastore.go` - Defines the implementation of the `ContactsDataAccessor` interface.
```go
package datastores
...

type ContactsDatastore struct {
	interfaces.DbHandler
}

func (ds *ContactsDatastore) ListUserContacts(userID string) ([]models.Contact, error) {
	... // Covered in the next article in this series.
}

func (ds *ContactsDatastore) CreateUserContact(userID string, contact models.Contact) (models.Contact, error) {
	... // Covered in the next article in this series.
}
```

----

## Summary
At this point you should have some Go code that defines a pretty good skeleton of the Contacts API. We have created a main application entry point that binds various API routes to HTTP handlers defined by a controller, and we have wired up the controller with the interfaces it needs to do it's job. We have also implemented a service struct that implements the `ContactsManager` interface, so we're one step closer to having a working application. In ["Go API Development: Part 2"](../go-api-dev-database-layer) we will implement the `ContactsDataAccessor` interface so that we can start to actually persist the Contacts that we manage.
