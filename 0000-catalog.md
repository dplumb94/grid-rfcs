- Feature Name: Catalog
- Start Date: 2019-07-12
- RFC PR: 
- Hyperledger Grid Issue: [GRID-95](https://jira.hyperledger.org/browse/GRID-95)

# Summary
[summary]: #summary

This RFC proposes a generic implementation for a Hyperledger Grid *Catalog*. The
implementation of *Catalog* will materialize as an assortment of grid products
that can be shared with one or more organizations, as well as implementaion of
catalog related functions. The Catalog on grid will contain 5 fields. A
**catalog_id**, a **catalog_owner**, an **expiry_date**, a **Name** for the 
catalog, and a repeated field of PropertyValues for custom fields.

In addition any grid product added to a catalog will need to contain a
***status*** field and can include an optional ***prices*** field, which will be 
enforced by a grid schema.

# Motivation
[motivation]: #motivation

The Grid Catalog implementation is designed for sharing an assortment of
products between organizations. Each product in a catalog will contain a number
of master data elements. Using the Grid Catalog, organizations can share
collections of related/unrelated products with other participants in their
network. 

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Entities

A **catalog** is a high level construct that products adhereing to the 
"catalog_product" schema can reference as an additional field.  A catalog can be 
referenced, or used to do business in a supply chain. A catalog is referenced by 
a "catalog_product" via its **catalog_id**. A catalog also has an **owner** (the 
organization that creates the catalog).  It also contains an **expiry_date** as 
a mechanism for clients to exipire the catalog. Also a **Name** for the catalog. 

A catalog, like a product, can also have one or more **properties**.  Properties
are described in the Grid Primitives RFC.  A *property namespace* contains
multiple *property schemas*.  A property schema associates a name (such as
"length") with a data type (such as integer).  

## Transactions

Catalogs are managed by submitting transactions to Hyperledger Grid, which will
process them with the Grid Catalog smart contract. The following transactions
are supported:

**catalog actions:**
* CatalogCreate - create a catalog and store it in state
* CatalogDelete - removes a catalog from state
* CatalogReplicate - creates a full copy of a catalog, or a sub catalog from 
catalog_products that already exist in state

**catalog_product operations:**
* AddProductsToCatalog - adds catalog_product(s) specified by their product_id 
to the catalog (as a list)
* RemoveProductsFromCatalog - removes catalog_product(s) from a catalog (as a list)
* ActivateProduct - toggles the "status" of a catalog_product to an active state
* DeactivateProduct - toggles the "status" of a catalog_product to an inactive state
* DiscontinueProduct - removes a catalog_product from all catalogs it's in

## Permissions

Creation of a Grid Catalog is restricted to agents acting on behalf of the
organization in the catalog's owner field.  An organization must have the GS1
company prefix in the "gs1_company_prefixes" metadata field for its Pike
organization.  (Organization-level metadata will need to be implemented in
Pike.) When a catalog is created, its owning organization is stored with the
catalog in an "owner" field.

Deletion of a catalog is restricted to agents acting on behalf of the
organization in the catalog's owner field.  A setting will turn off deletion
entirely, with the intent that it could be enabled/disabled by a Grid
administrator.  Disabling delete is useful because external systems may have
references to these products and deleting them could leave dangling references.

*Disclaimer: CatalogDelete is a potentially hazardous operation and needs to be 
done with care.*

Replication of a catalog is restricted to agents acting on behalf of an
organization. The purpose of replication is to enable agents to duplicate a 
catalog, or a subset of "catalog_products" in state with the intention of 
modifying the catalog_products to fit a different scheme. It reduces the 
overhead needed to recreate a catalog. Generally organizations have a single 
"master" catalog that they use to build "subcatalogs" customized for their 
business partners. 

To perform any of the **catalog actions** agents will need the following 
permissions:
```
CatalogCreate: Agent from org needs the "can_create_catalog" permission
CatalogDelete: Agent from org needs the "can_delete_catalog" permission
CatalogReplicate: Agent from org needs the "can_duplicate_catalog" permission
```

In the case of **catalog operations** agents from the org they represent will
need the operational permission to perform the desired operation on a Grid
Catalog.
```
AddProductsToCatalog: Agent from org needs the "can_add_products_to_catalog" permission
RemoveProductsFromCatalog: Agent from org needs the "can_remove_products_from_catalog" permission
ActivateProduct: Agent from org needs the "can_activate_product_in_catalog" permission
DeactivateProduct: Agent from org needs the "can_deactivate_product_in_catalog" permission
DiscontinueProduct: Agent from org needs the "can_discontinue_product_in_catalog" permission
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The primary object stored in state is **Catalog**, which consists of a
**catalog_id** ([UUID](https://crates.io/crates/uuid)), an **owner** (org_id 
compatible w/Pike), an **expiry_date** (unix timestamp), a **Name**, and a 
repeated field of **PropertyValues**.  The properties available are defined by 
the Grid Property Schema Transaction Family and are restricted to the fields and 
rules of the GS1 Catalog schema (which will be defined at a later time).  
Transactions which are responsible for setting product state values must ensure 
that those properties conform with the requirements of the GS1 Catalog Property 
Schema (to be defined at a later time).

``` 
message Catalog { 
    string catalog_id = 1; 
    string owner = 2; 
    string expiry_date = 3; 
    name = 4; 
    repeated PropertyValue properties = 6; 
} 
```

### Referencing a Catalog

Products are uniquely referenced by their product_id and product_namespace.  For
GS1, Products are referenced by the GTIN identifier. For example:

``` 
get_catalog(catalog_id) // UUID 
set_catalog(catalog_id, catalog) // UUID, catalog 
```

### Catalog Addressing in the Merkle-Radix State System

In order to uniquely locate Catalogs in the Merkle-Radix state system, an
address must be constructed which identifies the storage location of the Catalog
representation.

All Grid addresses are prefixed by the 6-hex-character namespace prefix
"621dee",  Catalogs are further prefixed under the Grid namespace with reserved
enumerations of "03" ("00", "01", and "02" being reserved for other purposes)
indicating "Catalogs" and an additional "01" indicating "GS1 Catalog".

Therefore, all addresses starting with:

``` 
"621dee" + "03" + "01" 
```

are Grid GS1 Catalogs identified by a UUID which can be reference by a 
"catalog_product" to group products to a specific catalog. 

UUID formats consist of 36-digit "alphanumeric strings" which include a fixed 
amount of internal "0" padding.  After the 10-hex-characters that are consumed 
by the grid namespace prefix, the catalog, and GS1 prefixes, there are 60 hex 
characters remaining in the address.  The 36 digits of the GTIN can be left 
padded with 24-hex-character zeroes and right padded with 2-hex-character zeroes 
to accommodate potential future storage associated with the GS1 Catalog
representation, for example:

``` 
"621dee" + "03" + "01" + 0000000000000000000000 +
36-character "numeric string" catalog_id + "00" // catalog_id == UUID 
```

Using the v5 constructer in the UUID crate we can tie the catalog_id to 
catalog_name will prevent duplication of catalogs when invoking the 
catalog_replicate action.  
```
use uuid::Uuid;

fn main() {
    let my_uuid = Uuid::new_v5(&Uuid::new_v4(), b"catalog_name");
    println!("{}", my_uuid);
}

>> 9db5fcec-dcab-5b04-b31b-765ccbf2bc3a
```

The full catalog address would look like:
``` 
"621dee030100000000000000000000009db5fcec-dcab-5b04-b31b-765ccbf2bc3a00" 
```


### Catalog Product Addressing in the Merkle-Radix State System

The Grid GS1 Catalog Product state address will be prefixed with the Grid 
namespace of "621dee", in addition to "02" (product namespace), "01" for (gs1 
product namespace), and and additional "01" to specify that it is a grid gs1 
catalog product namespace.

Therefore, all addresses starting with:

``` 
"621dee" + "02" + "01" + "01"
```

are Grid GS1 Catalog Products identified by a gtin and are expected to contain 
the catalog_id of the catalog they are associated with, as well as extra 
PropertyValues that are defined in the GS1 catalog product schema. These 
PropertyValues are in addition to the expected PropertyValues of a [Grid GS1 
Product](https://github.com/hyperledger/grid-rfcs/blob/fbedec06d70b16492fea9f6b1e87146c5fc56771/0000-product.md).

GTIN formats consist of 14-digit “numeric strings” which include some amount of
internal “0” padding depending on the specific GTIN format (GTIN-12,
GTIN-13, or GTIN-14).  After the 12-hex-characters that are consumed by the grid
namespace prefix, the product namespace prefix, GS1 namespce prefix, and catalog 
product namespace prefix, there are 58 hex characters remaining in the address.  
The 14 digits of the GTIN can be left padded with 42-hex-character zeroes and 
right padded with 2-hex-character zeroes to accommodate potential future storage 
associated with the GS1 Product
representation, for example:

``` 
“621dee” + “02” + “01” + "01" + “000000000000000000000000000000000000000000” 
+ 14-character “numeric string” product_id + “00” // product_id == GTIN 
```

A full GS1 Product address using the example GTIN from https://www.gtin.info/
would therefore be:

``` 
“621dee0201010000000000000000000000000000000000000000000001234560001200” 
```

# Rationale and alternatives
[alternatives]: #alternatives

The design of catalog in this manner leaves it extensible for domain specific
use. The concept of price has been discussed and it's been decided to include 
the notion of price within products added to a catalog. A simple implementation 
of price is fine and currently serves as a placeholder, while a more robust 
standard of modeling price data can be implemented from a future pricing-related
RFC. A GS1 pricing standard that can be leveraged in the writing of such an RFC
would be [GS1 Price 
Sync](https://www.gs1.org/docs/gdsn/3.1BMS_Price_Sync_r3p1p3_i1p3p5_23May2017.pdf).

Ideally the catalog_id should be dervied from the org_id and perhaps the catalog_name

# Prior art
[prior-art]: #prior-art

In the distributed ledger space there are not any example implementations of a
product catalog. There are specs and standards around catalogs that can be
leveraged in the design and implementation of a Grid Catalog.

See Ariba's® [catalog interchange format
(CIF)](https://www.essent.com/What-is-a-CIF-Catalog.html) CIF that has grown
into a de facto standard and is widely used by hundreds of thousands of
companies throughout the world.

[GS1's Catalogue Item 
Synchronisation](https://www.gs1.org/docs/gdsn/3.1/BMS_GDSN_Catalogue_Item_Sync_r3p1p0_i1_p0_p6_25Aug2015.pdf)
is the process of continuous harmonisation of item information between trading
partners which ensures that the master data is the same in all trading partners
systems.

CIF and GS1's CIS provides a view of current state of product catalogs as well 
as the formats and transactions associated with them.

# Unresolved questions
[unresolved]: #unresolved-questions

- Determining the integration with Grid Schema
- Determining schemas for catatlog products
- Determining pricing data model and associated function in a future RFC