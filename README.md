# Shopify - AusPost Rates

Australia Post zones and rate data for importing to Shopify.

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

You will need to get the delivery profile ID's and location group ID's to use in other queries.

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
```

Parametised queries use `application/json` instead of raw `application/graphql`, with the query passed as a string
parameter. 

You can view the content of a profile, for the zone definitions with the countries in that zone and the list of 
delivery methods and prices (for different methods or conditions).

Use this to examine your current profiles, or to check the contents after you have created a new profile for
Australia Post.

```pwsh
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
      [PSCustomObject]@{ zone = $zone.name; country = $null; countryName = $null; province = $null; provinceName = $null }
    } else {
      if (-not $country.provinces) {
        [PSCustomObject]@{ zone = $zone.name; country = $country.code.countryCode; countryName = $country.name; province = $null; provinceName = $null }
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

This can then be saved to a comma separated value (CSV) file, e.g. for manipulation in a spreadsheet program.

```
$zoneData | Export-Csv 'data/zone-country-province.csv'
```

## Generating rate data files


## Load Australia Post data to a shipping profile

Create a new profile named 'Australia Post' to load the data into.

Use the data files to create zones and assign countries to them, then load the shipping rates for the
zones.

The Shopify API reference for delivery profile updates is: 
https://shopify.dev/docs/admin-api/graphql/reference/shipping-and-fulfillment/deliveryprofileupdate

### Zones and countries

#### Read zone and country data

A zone-country CSV data file can be used to create zone information for input.

```
$zoneCountryData = Import-Csv 'data/auspost-zone-country-province.csv'
$zonesToCreate = [System.Collections.ArrayList]@()
$zoneInput = $null
$zoneCountryData | ForEach-Object {
  $line = $_
  if ($line.zone -ne $zoneInput.name) {
    $zoneInput = @{ name = $line.zone; countries = [System.Collections.ArrayList]@() }
    $countryInput = $null
    $i = $zonesToCreate.Add($zoneInput)
  }
  if (-not $line.country) {
    $i = $zoneInput.countries.Add(@{ restOfWorld = $true })
  } else {
    if ($line.country -ne $countryInput.code) {
      if ($line.province) {
        $countryInput = @{ code = $line.country; provinces = [System.Collections.ArrayList]@() }
      } else {
        $countryInput = @{ code = $line.country; includeAllProvinces = $true }
      }
      $i = $zoneInput.countries.Add($countryInput)
    }
    if ($line.province) {
      $i = $countryInput.provinces.Add(@{ code = $line.province })
    }
  }
}
$zonesToCreate | ConvertTo-Json -Depth 5
```

#### Load zones and countries to shipping profile

Within a profile, each location group that you ship from has different rates. In the example below there is only one
location group to update.

Get to profile to be updated, and add the zones to create to the profile location group ID.

```
$deliveryProfile = $deliveryProfiles.data.deliveryProfiles.edges | Where-Object { $_.node.name -eq 'Australia Post' }
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
      profileLocationGroups {
        locationGroupZones (first: 15) {
          edges {
            node {
              zone {
                id
                name
              }
              methodDefinitions (first:20) {
                edges {
                  node {
                    id
                    name
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

### Shipping rates

#### Read shipping rate data

To update the zones we have created with the rates, first we need to get the created zone IDs.

```pwsh
$getDeliveryProfileZonesQuery = 'query($id: ID!)
{
  deliveryProfile (id: $id) {
    profileLocationGroups {
      locationGroupZones (first: 15) {
        edges {
          node {
            zone {
              id
              name
            }
          }
        }
      }
    }
  }
}'

$getDeliveryProfileZonesData = @{
  query = $getDeliveryProfileZonesQuery;
  variables = @{
    id = $deliveryProfile.node.id;
  }
}

$profileZones = Invoke-RestMethod -Method Post -Uri $uri -Headers $jsonHeaders -Body (ConvertTo-Json -Depth 3 $getDeliveryProfileZonesData)
$zoneIdsAndNames = $profileZones.data.deliveryProfile.profileLocationGroups[0].locationGroupZones.edges.node.zone
$zoneIdsAndNames | Measure-Object
```

Read the data file and use it to build the zone updates adding the method definitions.

```pwsh
$zoneRateData = Import-Csv 'data/auspost-rates-to-1kg.csv'
$zonesToUpdate = [System.Collections.ArrayList]@()
$currentZone = $null
$zoneRateData | ForEach-Object {
  $line = $_
  if ($line.zone -ne $currentZone) {
    $currentZone = $line.zone
    $zone = $zoneIdsAndNames | Where-Object { $_.name -eq $currentZone}
    if (-not $zone) { throw "Zone $($_.name) not found" }
    $zoneInput = @{ id = $zone.id; methodDefinitionsToCreate = [System.Collections.ArrayList]@() }
    $i = $zonesToUpdate.Add($zoneInput)
  }
  $weightConditionsInput = [System.Collections.ArrayList]@()
  $i = $weightConditionsInput.Add(@{ criteria = @{ unit = 'KILOGRAMS'; value = [decimal]$line.lessThanKg; }; operator = 'LESS_THAN_OR_EQUAL_TO' })
  if ([decimal]$line.greaterThanKg) {
    $i = $weightConditionsInput.Add(@{ criteria = @{ unit = 'KILOGRAMS'; value = [decimal]$line.greaterThanKg; }; operator = 'GREATER_THAN_OR_EQUAL_TO' })
  }
  $methodInput = @{ 
    active = $true;
    name = $line.method;
    rateDefinition = @{ price = @{ amount = [decimal]$line.rateAud; currencyCode = 'AUD' } };
    weightConditionsToCreate = $weightConditionsInput;
  }
  $i = $zoneInput.methodDefinitionsToCreate.Add($methodInput)
}
$zonesToUpdate | ConvertTo-Json -Depth 6
```

#### Uploading rates

```
$updateProfileQuery = 'mutation($id: ID!, $profile: DeliveryProfileInput!) {
  deliveryProfileUpdate (id: $id, profile: $profile)
  {
    profile {
      id
      name
      profileLocationGroups {
        locationGroupZones (first: 15) {
          edges {
            node {
              zone {
                id
                name
              }
              methodDefinitions (first:20) {
                edges {
                  node {
                    id
                    name
                  }
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

$profileLocationGroupUpdateInput = @{ id = $locationGroupId; zonesToUpdate = $zonesToUpdate }

$addRates = @{
  query = $updateProfileQuery;
  variables = @{
    id = $deliveryProfile.node.id;
    profile = @{
      locationGroupsToUpdate = @( $profileLocationGroupUpdateInput )
    }
  }
}

$addRatesResult = Invoke-RestMethod -Method Post -Uri $uri -Headers $jsonHeaders -Body (ConvertTo-Json -Depth 11 $addRates)
$addRatesResult
```

### Replacing existing rates

To replace existing rates, you remove all the old method definitions for that profile,
and create new ones.

First get the existing rates. This uses the query from 'Get existing shipping profile information',
with the Australia Post delivery profile.

```
$getDeliveryProfileData = @{
  query = $getDeliveryProfileQuery;
  variables = @{
    id = $deliveryProfile.node.id;
  }
}

$profileDetails = Invoke-RestMethod -Method Post -Uri $uri -Headers $jsonHeaders -Body (ConvertTo-Json -Depth 3 $getDeliveryProfileData)
```

The list of existing delivery method definitions can be easily shown:

```
$methodsToDelete = $profileDetails.data.deliveryProfile.profileLocationGroups.locationGroupZones.edges.node.methodDefinitions.edges.node.id
$methodsToDelete
```

Then follow the instructions in 'Read shipping rate data' and 'Uploading rates' up to the point where
`$profileLocationGroupUpdateInput` is created.

Then use both `$profileLocationGroupUpdateInput` and `$methodsToDelete` to update the rates.

```
$updateRates = @{
  query = $updateProfileQuery;
  variables = @{
    id = $deliveryProfile.node.id;
    profile = @{
      methodDefinitionsToDelete = $methodsToDelete
      locationGroupsToUpdate = @( $profileLocationGroupUpdateInput )
    }
  }
}

$updateRatesResult = Invoke-RestMethod -Method Post -Uri $uri -Headers $jsonHeaders -Body (ConvertTo-Json -Depth 11 $updateRates)
$updateRatesResult
```

### Replacing existing zones

Get details of the existing profile you want to replace:

```
$getDeliveryProfileData = @{
  query = $getDeliveryProfileQuery;
  variables = @{
    id = $deliveryProfile.node.id;
  }
}

$profileDetails = Invoke-RestMethod -Method Post -Uri $uri -Headers $jsonHeaders -Body (ConvertTo-Json -Depth 3 $getDeliveryProfileData)
```

Get all existing location group zones:

```
$zonesToDelete = $profileDetails.data.deliveryProfile.profileLocationGroups.locationGroupZones.edges.node.zone.id
$zonesToDelete
```

Then delete them:

```
$deleteZones = @{
  query = $updateProfileQuery;
  variables = @{
    id = $deliveryProfile.node.id
    profile = @{
      zonesToDelete = $zonesToDelete
    }
  }
}

$deleteZonesResult = Invoke-RestMethod -Method Post -Uri $uri -Headers $jsonHeaders -Body (ConvertTo-Json -Depth 11 $deleteZones)
$deleteZonesResult
```

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
$activeProducts = $wholesaleProducts.data.products.edges.node | ? { $_.status -eq 'ACTIVE' }
$activeProducts | Measure-Object

$activeDeliveryProfileId = ($deliveryProfiles.data.deliveryProfiles.edges | Where-Object { $_.node.name -eq 'Australia Post' }).node.id

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

$addActiveProducts = @{
  query = $updateProfileQuery;
  variables = @{
    id = $activeDeliveryProfileId;
    profile = @{
      variantsToAssociate = $activeProducts.variants.edges.node.id
    }
  }
}

$addResult1 = Invoke-RestMethod -Method Post -Uri $uri -Headers $jsonHeaders -Body (ConvertTo-Json -Depth 4 $addActiveProducts)
$addResult1
```

You can reuse the same query with different variables:

```
$otherProducts = $wholesaleProducts.data.products.edges.node | ? { $_.status -ne 'ACTIVE' }
$otherProducts | Measure-Object

$otherDeliveryProfileId = ($deliveryProfiles.data.deliveryProfiles.edges | Where-Object { $_.node.name -eq 'Australia Post 2' }).node.id

$addOtherProducts = @{
  query = $updateProfileQuery;
  variables = @{
    id = $otherDeliveryProfileId;
    profile = @{
      variantsToAssociate = $otherProducts.variants.edges.node.id
    }
  }
}

$addResult2 = Invoke-RestMethod -Method Post -Uri $uri -Headers $jsonHeaders -Body (ConvertTo-Json -Depth 4 $addOtherProducts)
$addResult2.data.deliveryProfileUpdate.profile
```
