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


## Get existing shipping profile information

```pwsh
$body = '{
  deliveryProfiles(first:10) {
    edges {
      node {
        id
        name
        default
        profileLocationGroups {
          locationGroup {
            id
          }
        }
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

### Generate a table of existing zone definitions

This outputs a summary of the zone name, and countries and provinces allocated to that zone.

```
$zoneData = $defaultProfileDetails.data.deliveryProfile.profileLocationGroups[0].locationGroupZones.edges | ForEach-Object {
  $zone = $_.node.zone
  $zone.countries | ForEach-Object {
    $country = $_
    if ($country.code.restOfWorld) {
      [PSCustomObject]@{ zone = $zone.name; country = $empty; countryName = $empty; province = $empty; provinceName = $empty }
    } else {
      if (-not $country.provinces) {
        [PSCustomObject]@{ zone = $zone.name; country = $country.code.countryCode; countryName = $country.name; province = $empty; provinceName = $empty }
      } else {;
        $country.provinces | ForEach-Object {
          $province = $_
          [PSCustomObject]@{ zone = $zone.name; country = $country.code.countryCode; countryName = $country.name; province = $province.code; provinceName = $province.name }
        }
      }
    }
  }
}
$zoneData | Format-Table
```

This can then be saved to a comma separated value (CSV) file, e.g. for manipulation on a spreadsheet program.

```
$zoneData | Export-Csv 'data/zone-country-province.csv'
```

## Generating rate data files


## Creating input structure from data

A zone-country CSV data file can be used to create zone information for input.

```
$zoneCountryData = Import-Csv 'data/auspost-zone-country-province.csv'
$zonesToCreate = [System.Collections.ArrayList]@()
$zoneCountryData | ForEach-Object {
  $line = $_
  if ($line.zone -ne $zoneInput.name) {
    $zoneInput = @{ name = $line.zone; countries = [System.Collections.ArrayList]@() }
    $countryInput = $empty
    $i = $zonesToCreate.Add($zoneInput)
  }
  if (-not $line.country) {
    $zoneInput.countries.Add(@{ restOfWorld = $true })
  } else {
    if ($line.country -ne $countryInput.code) {
      if ($line.province) {
        $countryInput = @{ code = $line.country; provinces = [System.Collections.ArrayList]@() }
      } else {
        $countryInput = @{ code = $line.country; includeAllProvinces = $true }
      }
      $zoneInput.countries.Add($countryInput)
    }
    if ($line.province) {
      $countryInput.provinces.Add(@{ code = $line.province })
    }
  }
}
$zonesToCreate | ConvertTo-Json -Depth 5
```


## Uploading zones

Within a profile, each location group that you ship from has different rates. In the example below there is only one
location group to update.

Get to profile to be updated, and add the zones to create to the profile location group ID.

```
$deliveryProfile = $deliveryProfiles.data.deliveryProfiles.edges | Where-Object { $_.node.name -eq 'Wholesale Shipping' }
$deliveryProfile.node.profileLocationGroups | Measure-Object
$locationGroupId = $deliveryProfile.node.profileLocationGroups[0].locationGroup.id
$profileLocationGroupInput = @{ id = $locationGroupId; zonesToCreate = $zonesToCreate }
```

Send this as an update.

```
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

$addZones = @{
  query = $updateProfileQuery;
  variables = @{
    id = $deliveryProfile.node.id;
    profile = @{
      locationGroupsToUpdate = @( $profileLocationGroupInput )
    }
  }
}

$addZonesResult = Invoke-RestMethod -Method Post -Uri $uri -Headers $jsonHeaders -Body (ConvertTo-Json -Depth 10 $addZones)
$addZonesResult
```

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
