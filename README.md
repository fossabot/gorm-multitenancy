# gorm-multitenancy

[![Go Reference](https://pkg.go.dev/badge/github.com/bartventer/gorm-multitenancy.svg)](https://pkg.go.dev/github.com/bartventer/gorm-multitenancy/v2)
[![Go Report Card](https://goreportcard.com/badge/github.com/bartventer/gorm-multitenancy)](https://goreportcard.com/report/github.com/bartventer/gorm-multitenancy)
[![Coverage Status](https://coveralls.io/repos/github/bartventer/gorm-multitenancy/badge.svg?branch=master)](https://coveralls.io/github/bartventer/gorm-multitenancy?branch=master)
[![Build](https://github.com/bartventer/gorm-multitenancy/actions/workflows/go.yml/badge.svg)](https://github.com/bartventer/gorm-multitenancy/actions/workflows/go.yml)
[![License](https://img.shields.io/github/license/bartventer/gorm-multitenancy.svg)](LICENSE)
[![FOSSA Status](https://app.fossa.com/api/projects/git%2Bgithub.com%2Fbartventer%2Fgorm-multitenancy.svg?type=shield)](https://app.fossa.com/projects/git%2Bgithub.com%2Fbartventer%2Fgorm-multitenancy?ref=badge_shield)

There are three common approaches to multitenancy in a database:
- Shared database, shared schema
- Shared database, separate schemas
- Separate databases

This package implements the shared database, separate schemas approach. It uses the [gorm](https://gorm.io/) ORM to manage the database and provides custom drivers to support multitenancy. It also provides HTTP middleware to retrieve the tenant from the request and set the tenant in context.

## Database compatibility
Current supported databases are listed below. Pull requests for other drivers are welcome.
- [PostgreSQL](https://www.postgresql.org/)

## Router compatibility
Current supported routers are listed below. Pull requests for other routers are welcome.
- [echo](https://echo.labstack.com/docs)
- [net/http](https://golang.org/pkg/net/http/)

## Installation

```bash
go get -u github.com/bartventer/gorm-multitenancy/v2
```

## Usage

### PostgreSQL driver

#### Important notes
- The driver uses the `public` schema for public models and the tenant specific schema for tenant specific models
- All models must implement the `gorm.Tabler` interface
- The table name for public models must be prefixed with `public.` (e.g. `public.books`), whereas the table name for tenant specific models must not contain any prefix (e.g. only `books`)
- All tenant specific models must implement the [TenantTabler](https://pkg.go.dev/github.com/bartventer/gorm-multitenancy/v2/#TenantTabler) interface, which classifies the model as a tenant specific model:
    - The `TenantTabler` interface has a single method `IsTenantTable() bool` which returns `true` if the model is tenant specific and `false` otherwise
    - The `TenantTabler` interface is used to determine which models to migrate when calling `MigratePublicSchema` or `CreateSchemaForTenant`
- Models can be registered in two ways:
    - When creating the dialect, by passing the models as variadic arguments to `postgres.New` (e.g. `postgres.New(postgres.Config{...}, &Book{}, &Tenant{})`) or by calling `postgres.Open` (e.g. `postgres.Open("postgres://...", &Book{}, &Tenant{})`)
    - By calling `postgres.RegisterModels` (e.g. `postgres.RegisterModels(db, &Book{}, &Tenant{})`)
- Migrations can be performed in two ways (after registering the models):
    - By calling `postgres.MigratePublicSchema` to create the public schema and migrate all public models
    - By calling `postgres.CreateSchemaForTenant` to create the schema for the tenant and migrate all tenant specific models
- To drop a tenant schema, call `postgres.DropSchemaForTenant`; this will drop the schema and all tables in the schema

#### Foregin key constraints between public and tenant specific models
- Conforming to the [notes above](#important-notes), foreign key constraints between public and tenant specific models can be created just as if you were using approach 1 (shared database, shared schema).
- The easiest way to get this working is to embed the [postgres.TenantModel](https://pkg.go.dev/github.com/bartventer/gorm-multitenancy/v2/drivers/postgres#TenantModel) struct in your tenant model. This will add the necessary fields for the tenant model (e.g. `DomainURL` and `SchemaName`), you can then create a foreign key constraint between the public and tenant specific models using the `SchemaName` field as the foreign key (e.g. `gorm:"foreignKey:TenantSchema;references:SchemaName"`); off course, you can also create foreign key constraints between any other fields in the models.

#### Brief example of the main concepts (for a complete example refer to the [examples](#examples) section)
```go

import (
    "gorm.io/gorm"
    "github.com/bartventer/gorm-multitenancy/v2/drivers/postgres"
)

// For models that are tenant specific, ensure that TenantTabler is implemented
// This classifies the model as a tenant specific model when performing subsequent migrations

// Tenant is a public model
type Tenant struct {
    gorm.Model // Embed the gorm.Model
    postgres.TenantModel // Embed the TenantModel
}

// Implement the gorm.Tabler interface
func (t *Tenant) TableName() string {return "public.tenants"} // Note the public. prefix

// Book is a tenant specific model
type Book struct {
    gorm.Model // Embed the gorm.Model
    Title string

    // FK to TenantSchema (same as if you were using approach 1; not realy needed if you use 
    // approach 2 as the schema is constrained to the tenant already, but included to show how 
    // to create foreign key constraints between public a tenant specific models)
    TenantSchema string `gorm:"column:tenant_schema"`
	Tenant       Tenant `gorm:"foreignKey:TenantSchema;references:SchemaName"`

}

// Implement the gorm.Tabler interface
func (b *Book) TableName() string {return "books"} // Note the lack of prefix

// Implement the TenantTabler interface
func (b *Book) IsTenantTable() bool {return true} // This classifies the model as a tenant specific model

func main(){
    // Create the database connection
    db, err := gorm.Open(postgres.New(postgres.Config{
        DSN:                  "host=localhost user=postgres password=postgres dbname=postgres port=5432 sslmode=disable",
    }), &gorm.Config{})
    if err != nil {
        panic(err)
    }

    // Register the models
    // Models are categorized as either public or tenant specific, which allow for simpler migrations
    if err := postgres.RegisterModels(
        db,        // Database connection
        // Public models (does not implement TenantTabler or implements TenantTabler with IsTenantTable() returning false)
        &Tenant{},  
        // Tenant specific model (implements TenantTabler)
        &Book{},
        ); err != nil {
        panic(err)
    }

    // Migrate the database
    // Calling AutoMigrate won't work, you must either call MigratePublicSchema or CreateSchemaForTenant
    // MigratePublicSchema will create the public schema and migrate all public models
    // CreateSchemaForTenant will create the schema for the tenant and migrate all tenant specific models

    // Migrate the public schema (migrates all public models)
    if err := postgres.MigratePublicSchema(db); err != nil {
        panic(err)
    }

    // Create a tenant
    tenant := &Tenant{
        TenantModel: postgres.TenantModel{
            DomainURL: "tenant1.example.com",
            SchemaName: "tenant1",
        },
    }
    if err := db.Create(tenant).Error; err != nil {
        panic(err)
    }

    // Migrate the tenant schema
    // This will create the schema and migrate all tenant specific models
    if err := postgres.CreateSchemaForTenant(db, tenant.SchemaName); err != nil {
        panic(err)
    }

    // Operations on tenant specific schemas (e.g. CRUD operations on books)
    // Refer to Examples section for more details on how to use the middleware

    // Drop the tenant schema
    // This will drop the schema and all tables in the schema
    if err := postgres.DropSchemaForTenant(db, tenant.SchemaName); err != nil {
        panic(err)
    }

    // ... other operations
}
```

### echo middleware
For a complete example refer to the [PostgreSQL with echo](https://github.com/bartventer/gorm-multitenancy/tree/master/internal/examples/echo) example.
```go
import (
    multitenancymw "github.com/bartventer/gorm-multitenancy/v2/middleware/echo"
    "github.com/bartventer/gorm-multitenancy/v2/scopes"
    "github.com/labstack/echo/v4"
    // ...
)

func main(){
    // ...
    e := echo.New()
    // ... other middleware
    // Add the multitenancy middleware
    e.Use(multitenancymw.WithTenant(multitenancymw.WithTenantConfig{
        DB: db,
        Skipper: func(r *http.Request) bool {
			return strings.HasPrefix(r.URL.Path, "/tenants") // skip tenant routes
		},
        TenantGetters: multitenancymw.DefaultTenantGetters,
    }))
    // ... other middleware

    // ... routes
    e.GET("/books", func(c echo.Context) error {
        // Get the tenant from context
        tenant, _ := multitenancymw.TenantFromContext(c)
        var books []Book
        // Query the tenant specific schema
        if err := db.Scopes(scopes.WithTenant(tenant)).Find(&books).Error; err != nil {
            return echo.NewHTTPError(http.StatusInternalServerError, err.Error())
        }
        return c.JSON(http.StatusOK, books)
    })

    // ... rest of code
}

```

### net/http middleware
For a complete example refer to the [PostgreSQL with net/http](https://github.com/bartventer/gorm-multitenancy/tree/master/internal/examples/nethttp) example.
```go
import (
    "encoding/json"
    "net/http"

    "github.com/go-chi/chi/v5"
    multitenancymw "github.com/bartventer/gorm-multitenancy/v2/middleware/nethttp"
    "github.com/bartventer/gorm-multitenancy/v2/scopes"
    // ...
)

func main(){
    // ...
    r := chi.NewRouter() // your router of choice
    // ... other middleware
    // Add the multitenancy middleware
    r.Use(multitenancymw.WithTenant(multitenancymw.WithTenantConfig{
        DB: db,
        Skipper: func(r *http.Request) bool {
            return strings.HasPrefix(r.URL.Path, "/tenants") // skip tenant routes
        },
        TenantGetters: multitenancymw.DefaultTenantGetters, 
    }))
    // ... other middleware

    // ... routes
    r.Get("/books", func(w http.ResponseWriter, r *http.Request) {
        // Get the tenant from context
        tenant, _ := multitenancymw.TenantFromContext(r.Context())
        var books []Book
        // Query the tenant specific schema
        if err := db.Scopes(scopes.WithTenant(tenant)).Find(&books).Error; err != nil {
            http.Error(w, err.Error(), http.StatusInternalServerError)
            return
        }
        if err := json.NewEncoder(w).Encode(books); err != nil {
            http.Error(w, err.Error(), http.StatusInternalServerError)
            return
        }
    })

    // ... rest of code
}

```


## Examples

- [PostgreSQL with echo](https://github.com/bartventer/gorm-multitenancy/tree/master/internal/examples/echo)
- [PostgreSQL with net/http](https://github.com/bartventer/gorm-multitenancy/tree/master/internal/examples/nethttp)

## License

This project is licensed under the Apache License 2.0 - see the [LICENSE](LICENSE) file for details.


[![FOSSA Status](https://app.fossa.com/api/projects/git%2Bgithub.com%2Fbartventer%2Fgorm-multitenancy.svg?type=large)](https://app.fossa.com/projects/git%2Bgithub.com%2Fbartventer%2Fgorm-multitenancy?ref=badge_large)

## Contributing

All contributions are welcome! Open a pull request to request a feature or submit a bug report.