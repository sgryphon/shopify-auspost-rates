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

$defaultProfileId = ((Invoke-RestMethod -Method Post -Uri $uri -Headers $headers -Body $body).data.deliveryProfiles.edges | Where-Object { $_.node.default }).node.id

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

$r1 = Invoke-RestMethod -Method Post -Uri $uri -Headers $jsonHeaders -Body (ConvertTo-Json -Depth 3 $getDeliveryProfileData)
$r1 | ConvertTo-Json -Depth 15

```


## Generating rate data files


## Uploading rates


## Assigning products to profiles



