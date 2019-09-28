---
title: "Testing with Dependency Injection in Go using GoMock"
date: 2019-09-28
categories: ["golang", "go", "gomock", "dependency-injection", "dependencies", "mock"]
---

## Overview
In this article I will demonstrate how to use the [golang/gomock](https://github.com/golang/mock) project to generate mock implementations of interfaces. The mock implementations will be used to inject behavior into the dependencies of a Go app for testing.

<!--more-->

----

To continue along with the theme of previous articles, we will continue to work with a Contacts Manager API. Recall in ["Go API Development: Part 1"](../go-api-dev-routes/) and ["Go API Development: Part 2"](../go-api-dev-database-layer)  we have broken down the API implementation using the Go service pattern.

We're dealing with four primary interfaces: an interface for the service layer (where the business logic of the API resides), an interface for the data access layer that the service layer uses to interact with the database, and the final two interfaces are to abstract away the low-level details of the underlying database technology.

[`interfaces/ContactsManager.go`](https://github.com/jon-whit/go-contacts/blob/master/interfaces/ContactsManager.go) - Defines the interface to manage Contacts (service layer).
```go
type ContactsManager interface {
	ListUserContacts(userID string) ([]models.Contact, error)
	CreateContact(userID string, contact models.Contact) (models.Contact, error)
}
```

[`interfaces/ContactsDataAccessor.go`](https://github.com/jon-whit/go-contacts/blob/master/interfaces/ContactsDataAccessor.go) - Defines the data access layer interface to manage the persistence of Contact information (datastore layer).
```go
type ContactsDataAccessor interface {
	ListUserContacts(userID string) ([]models.Contact, error)
	CreateUserContact(userID string, contact models.Contact) (models.Contact, error)
}
```

[`interfaces/DBHandler.go`](https://github.com/jon-whit/go-contacts/blob/master/interfaces/DBHandler.go) - Defines the interfaces to interact with a SQL database backend.
```go
type DBHandler interface {
	Query(query string, args ...interface{}) (DBRow, error)
	Execute(query string, args ...interface{}) (sql.Result, error)
}

type DBRow interface {
	Scan(dest ...interface{}) error
	Next() bool
}
```

## Building Mocks
To build mock impelmentations of the interfaces above we use the [golang/gomock](https://github.com/golang/mock) tool called `mockgen`.

### Installing mockgen
Installing the `mockgen` tool is as simple as:

`go get github.com/golang/mock/mockgen`

### Running mockgen
To generate mock interfaces from a source file point `mockgen` to the file and specify the output destination of the generated mock and the package for the mocked interface. To generate the mocks for the interfaces above run the following commands:

```go
mockgen -source interfaces/ContactsManager.go -destination mocks/mock_ContactsManager.go -package mocks ContactsManager
```

```go
mockgen -source interfaces/ContactsDataAccessor.go -destination mocks/mock_ContactsDataAccessor.go -package mocks ContactsDataAccessor
```

```go
mockgen -source interfaces/DBHandler.go -destination mocks/mock_DBHandler.go -package mocks DBHandler
```

For more information see the [official documentation](https://github.com/golang/mock#running-mockgen) page.

### Autogenerate Mocks
The commands above that were used to generate mocks of the interfaces can be included as `go generate` directives in the  source files themselves. This will ensure the mocked interfaces are kept up to date with changes to the interface definitions. To autogenerate mocks, add a comment including a `go:generate` directive at the top of each of the files defining the interfaces.

[`interfaces/ContactsManager.go`](https://github.com/jon-whit/go-contacts/blob/master/interfaces/ContactsManager.go)
```go
//go:generate mockgen -source ../interfaces/ContactsManager.go -destination ../mocks/mock_ContactsManager.go -package mocks ContactsManager

package interfaces
...
```

[`interfaces/ContactsDataAccessor.go`](https://github.com/jon-whit/go-contacts/blob/master/interfaces/)
```go
//go:generate mockgen -source ../interfaces/ContactsDataAccessor.go -destination ../mocks/mock_ContactsDataAccessor.go -package mocks ContactsDataAccessor

package interfaces
...
```

[`interfaces/DBHandler.go`](https://github.com/jon-whit/go-contacts/blob/master/interfaces/DBHandler.go)
```go
//go:generate mockgen -source ../interfaces/DBHandler.go -destination ../mocks/mock_DBHandler.go -package mocks DBHandler
  
package interfaces
...
```

Running `go generate ./...` will now automatically generate the mock interfaces and output the mocked interfaces in the `mocks/` directory.


## Testing with the Mocked Interfaces

In [Go API Development: Part 1](../go-api-dev-routes/) we defined the [`ContactsService`](https://github.com/jon-whit/go-contacts/blob/5f9ca5b2ef34a76d5d627aef1e56c1942a04f4ef/services/ContactsService.go#L8-L10) struct which implements the [`ContactsManager`](https://github.com/jon-whit/go-contacts/blob/5f9ca5b2ef34a76d5d627aef1e56c1942a04f4ef/interfaces/ContactsManager.go#L7-L10) interface. 

```go
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

In our tests for the `ContactsService` we want to mock the response behavior of the `ContactsDataAccessor` so that we can make the proper assertion(s) on the code paths throughout the `ContactsService` implemenation. To do that we can instantiate a `MockContactsDataAccessor` that we generated earlier, and pass that into the `ContactsService` when we instantiate it in our tests.

For example:

```go
package services

import (
    "github.com/golang/mock/gomock"
    "github.com/jon-whit/go-contacts/mocks"
    ...
)

func TestListUserContacts(t *testing.T) {
	ctrl := gomock.NewController(t)
	defer ctrl.Finish()

	// Create a mock ContactsDataAccessor
	mockDataAccessor := mocks.NewMockContactsDataAccessor(ctrl)

	// Specify the behavior of the mocked interface - Return an error from the database layer
	mockDataAccessor.EXPECT().ListUserContacts("1").Return(nil, errors.New("DB error"))

	s := ContactsService{
		DataAccessor: mockDataAccessor,
	}

	// Use the ContactsService to make make assertions on the ListUserContacts behavior
	contacts, err := s.ListUserContacts("1")

	assert.ElementsMatch(t, contacts, nil)
	assert.EqualError(t, err, "DB error")
}
```

When the `ListUserContacts` method of the `ContactsService` is invoked, the mocked behavior of the `ListUserContacts` method of the `ContactsDataAccessor` is executed. In this case, a nil slice of Contacts are returned and a database error is returned to mimic an error within the datastore layer. Appropriate assertions are made on the response behavior of the `ContactsService` based on the mocked behavior of the datastore layer.

---

If we change the the expectations on the mocked call to the `ListUserContacts` method of the `ContactsDataAccessor` so that the `userID` passed to it doesn't match with the `userID` that is passed down from the service layer


```go
package services
...

func TestListUserContacts(t *testing.T) {
    ...

	// Expect ListUserContacts to be called with userID "2"
	mockDataAccessor.EXPECT().ListUserContacts("2").Return(nil, errors.New("DB error"))

	s := ContactsService{
		DataAccessor: mockDataAccessor,
	}

	// ListUserContacts with userID "1"
	contacts, err := s.ListUserContacts("1")

	assert.ElementsMatch(t, contacts, nil)
	assert.EqualError(t, err, "DB error")
}
```

then we get the following error:

```go
--- FAIL: TestListUserContacts (0.00s)
    .../services/ContactsService.go:13: Unexpected call to *mocks.MockContactsDataAccessor.ListUserContacts([1]) at .../mocks/mock_ContactsDataAccessor.go:39 because: 
        Expected call at .../services/ContactsService_test.go:20 doesn't match the argument at index 0.
        Got: 1
        Want: is equal to 2
    asm_amd64.s:522: missing call(s) to *mocks.MockContactsDataAccessor.ListUserContacts(is equal to 2) .../services/ContactsService_test.go:20
    asm_amd64.s:522: aborting test due to missing call(s)
FAIL
FAIL	github.com/jon-whit/go-contacts/services	0.013s
```

By using the [golang/gomock](https://github.com/golang/mock) package we can achieve a high degree of confidence in our implementation of the Contacts Manager API because we can inject the behavior of dependencies between the various layers within our application, and we can make assertions on the contracts between these various layers as well.

We could also mock the `DBHandler` interface dependency of the [`ContactsDatastore`](https://github.com/jon-whit/go-contacts/blob/5f9ca5b2ef34a76d5d627aef1e56c1942a04f4ef/datastores/ContactsDatastore.go#L8-L10), which implements the `ContactsDataAccessor` interface, but I'll leave that exercise for you :thumbsup:. Please leave a comment below with a test or two for the [`ContactsDatastore`](https://github.com/jon-whit/go-contacts/blob/5f9ca5b2ef34a76d5d627aef1e56c1942a04f4ef/datastores/ContactsDatastore.go#L8-L10).