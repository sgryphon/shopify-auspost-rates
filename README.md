# Shopify - AusPost Rates

Zones an rate data for importing to Shopify.

## Requirements

This project contains PowerShell scripts that can be used to upload shipping rates to Shopify via the API.

You will need to have PowerShell Core (crossplatform) installed to run the scripts.

## Configure a Private App to get access to the API

In the Shopfiy admin, go to Apps > Manage private apps.

You will need to enable private apps, and then create a new app (call it something like "AusPost Rates" or "Data Manager", or whatever name makes sense to you). It will need Read and write permission to Shipping, as well as other functions you want to use, e.g. to
use scripts to add productsd to shipping, you need Read access to Products.

Record the API parameters in variables, which will be used in other scripts.

```pwsh
$passwordToken = '<Password>'
$shopName = '<shop name>'
```

## Accessing the GraphQL API

First set up the base URL and headers, using the variables above.

```pwsh
$uri = "https://$shopName.myshopify.com/admin/api/2021-01/graphql.json"
$headers = @{ 
  'Content-Type' = 'application/graphql';
  'X-Shopify-Access-Token' = $passwordToken
}
```

A simple query can be used to check the existing shipping profiles.

```pwsh
$body = '{
  deliveryProfiles(first:10) {
    edges {
      node {
        id
        name
        default
      }
    }
  }
}'

Invoke-RestMethod -Method Post -Uri $uri -Headers $headers -Body $body | ConvertTo-Json -Depth 5
```

You can also interactively test out queries in the Shopify API developer documentation site: 
https://shopify.dev/docs/admin-api/graphql/reference/shipping-and-fulfillment/deliveryprofile#samples


## Get existing shipping rates

```pwsh
$body = '{
  deliveryProfiles(first:10) {
    edges {
      node {
        id
        name
        default
      }
    }
  }
}'

$deliveryProfiles = Invoke-RestMethod -Method Post -Uri $uri -Headers $headers -Body $body
$defaultProfileId = ($deliveryProfiles.data.deliveryProfiles.edges | Where-Object { $_.node.default }).node.id

$jsonHeaders = @{ 
  'Content-Type' = 'application/json';
  'X-Shopify-Access-Token' = $passwordToken
}

$getDeliveryProfileQuery = 'query($id: ID!)
{
  deliveryProfile (id: $id) {
    profileLocationGroups {
      locationGroupZones (first: 20) {
        edges {
          node {
            zone {
              id
              name
                countries {
                  id
                  name
                  code {
                    countryCode
                    restOfWorld
                  }
                  provinces {
                    id
                    code
                    name
                  }
                }
            }
            methodDefinitions (first:8) {
              edges {
                node {
                  id
                  name
                  active
                  methodConditions {
                    conditionCriteria {
                      ... on Weight {
                        unit
                        value
                      }
                    }
                    field
                    operator
                  }
                  rateProvider {
                    ... on DeliveryRateDefinition {
                      id
                      price {
                        currencyCode
                        amount
                      }
                    }
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}'

$getDeliveryProfileData = @{
  query = $getDeliveryProfileQuery;
  variables = @{
    id = $defaultProfileId;
  }
}

$defaultProfileDetails = Invoke-RestMethod -Method Post -Uri $uri -Headers $jsonHeaders -Body (ConvertTo-Json -Depth 3 $getDeliveryProfileData)
$defaultProfileDetails | ConvertTo-Json -Depth 15

```


## Generating rate data files


## Uploading rates


## Assigning products to profiles

Get a list of all products you want to assign, e.g. from one vendor.

```
$getProductsQuery = 'query($first: Int, $filter: String)
  {
    products(first: $first, query: $filter) {
      edges {
        node {
          id
          handle
          vendor
          status
          variants (first: 2) {
            edges {
              node {
                id
                title
              }
            }
          }
        }
      }
      pageInfo {
        hasNextPage
      }
    }
  }
  '

$getProductsData = @{
  query = $getProductsQuery;
  variables = @{
    first = 150;
    filter = 'vendor:"Wholesale (AU)"'
  }
}

$wholesaleProducts = Invoke-RestMethod -Method Post -Uri $uri -Headers $jsonHeaders -Body (ConvertTo-Json -Depth 3 $getProductsData)
$wholesaleProducts.data.products.edges.node | Measure-Object
```

You can further filter the objects based on properties.

Then use an update query to add them to a delivery profile.

```
$inactiveProducts = $wholesaleProducts.data.products.edges.node | ? { $_.status -ne 'ACTIVE' }
$inactiveProducts | Measure-Object

$inactiveDeliveryProfileId = ($deliveryProfiles.data.deliveryProfiles.edges | Where-Object { $_.node.name -eq 'Wholesale Shipping 2' }).node.id

$updateProfileQuery = 'mutation($id: ID!, $profile: DeliveryProfileInput!) {
  deliveryProfileUpdate (id: $id, profile: $profile)
  {
    profile {
      id
      name
      profileItems (first: 150) {
        edges {
          node {
            product {
              id
              handle
              vendor
            }
            variants (first: 2) {
              edges {
                node {
                  id
                  title
                }
              }
            }
          }
        }
      }
    }
    userErrors {
      field
      message
    }
  }
}'

$addInactiveProducts = @{
  query = $updateProfileQuery;
  variables = @{
    id = $inactiveDeliveryProfileId;
    profile = @{
      variantsToAssociate = $inactiveProducts.variants.edges.node.id
    }
  }
}

$addResult1 = Invoke-RestMethod -Method Post -Uri $uri -Headers $jsonHeaders -Body (ConvertTo-Json -Depth 4 $addInactiveProducts)
$addResult1
```

You can reuse the same query with different variables:

```
$activeProducts = $wholesaleProducts.data.products.edges.node | ? { $_.status -eq 'ACTIVE' }
$activeProducts | Measure-Object

$activeDeliveryProfileId = ($deliveryProfiles.data.deliveryProfiles.edges | Where-Object { $_.node.name -eq 'Wholesale Shipping' }).node.id

$addActiveProducts = @{
  query = $updateProfileQuery;
  variables = @{
    id = $activeDeliveryProfileId;
    profile = @{
      variantsToAssociate = $activeProducts.variants.edges.node.id
    }
  }
}

$addResult2 = Invoke-RestMethod -Method Post -Uri $uri -Headers $jsonHeaders -Body (ConvertTo-Json -Depth 4 $addActiveProducts)
$addResult2.data.deliveryProfileUpdate.profile
```
