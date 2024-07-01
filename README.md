Shopify - AusPost Rates
=======================

Australia Post zones and rate data for importing to Shopify.


Requirements
------------

This project contains PowerShell scripts that can be used to upload shipping rates to Shopify via the API.

You will need to have PowerShell Core (crossplatform) installed to run the scripts.


Configure a Private App to get access to the API
------------------------------------------------

In the Shopfiy admin, go to Apps > Manage private apps.

You will need to enable private apps, and then create a new app (call it something like "AusPost Rates" or "Data Manager", or whatever name makes sense to you). It will need Read and write permission to Shipping, as well as other functions you want to use, e.g. to use scripts to add products to shipping, you need Read access to Products.

Record the API parameters in variables, which will be used in other scripts. For graph queries you will need the 'Password' value (you don't need the API key for Shared Secret).

```powershell
$password = '<Password>'
$shopName = '<shop name>'
```


Accessing the GraphQL API
-------------------------

First set up the base URL and headers, using the variables above.

```powershell
$uri = "https://$shopName.myshopify.com/admin/api/2021-01/graphql.json"
$headers = @{ 
  'Content-Type' = 'application/graphql';
  'X-Shopify-Access-Token' = $password
}
```

A simple query can be used to check the existing shipping profiles.

```powershell
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


Get existing shipping profile information
-----------------------------------------

You will need to get the delivery profile ID's and location group ID's to use in other queries.

```powershell
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
$defaultProfileId
```

Parametised queries use `application/json` instead of raw `application/graphql`, with the query passed as a string
parameter. 

You can view the content of a profile, for the zone definitions with the countries in that zone and the list of 
delivery methods and prices (for different methods or conditions).

Use this to examine your current profiles, or to check the contents after you have created a new profile for
Australia Post.

```powershell
$jsonHeaders = @{ 
  'Content-Type' = 'application/json';
  'X-Shopify-Access-Token' = $password
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

```powershell
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

```powershell
$zoneData | Export-Csv 'data/zone-country-province.csv'
```


Generating rate data files
--------------------------

### Load Australia Post data to a shipping profile

Create a new profile named 'Australia Post' to load the data into.

Use the data files to create zones and assign countries to them, then load the shipping rates for the
zones.

The Shopify API reference for delivery profile updates is: 
https://shopify.dev/docs/admin-api/graphql/reference/shipping-and-fulfillment/deliveryprofileupdate

### Select the profile to update

Get the profile you want to update based on the name:

```powershell
$deliveryProfile = $deliveryProfiles.data.deliveryProfiles.edges | Where-Object { $_.node.name -eq 'General Profile' }
$deliveryProfile.node.profileLocationGroups | Measure-Object
$locationGroupId = $deliveryProfile.node.profileLocationGroups[0].locationGroup.id
```

### Zones and countries

#### Read zone and country data

A zone-country CSV data file can be used to create zone information for input.

```powershell
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

Use the profile selected above, and add the zones to create to the profile location group ID.

```powershell
$profileLocationGroupInput = @{ id = $locationGroupId; zonesToCreate = $zonesToCreate }
```

Send this as an update.

```powershell
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

```powershell
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

```powershell
$zoneRateData = Import-Csv 'data/auspost-rates-to-5kg.csv'
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
  if ([decimal]$line.lessThanKg) {
    $i = $weightConditionsInput.Add(@{ criteria = @{ unit = 'GRAMS'; value = [decimal]$line.lessThanKg * 1000; }; operator = 'LESS_THAN_OR_EQUAL_TO' })
  }
  if ([decimal]$line.greaterThanKg) {
    $i = $weightConditionsInput.Add(@{ criteria = @{ unit = 'GRAMS'; value = [decimal]$line.greaterThanKg * 1000; }; operator = 'GREATER_THAN_OR_EQUAL_TO' })
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

```powershell
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
              methodDefinitions (first:40) {
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
```

If you are replacing rates continue back in the replacing instructions (see below); otherwise continue to add the new rates.

```powershell
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

Get the existing profile ID, based on the name (change the name to update different profiles)

```powershell
$deliveryProfile = $deliveryProfiles.data.deliveryProfiles.edges | Where-Object { $_.node.name -eq 'General Profile' }
```

First get the existing rates. This uses the query from 'Get existing shipping profile information',
with the Australia Post delivery profile.

```powershell
$getDeliveryProfileData = @{
  query = $getDeliveryProfileQuery;
  variables = @{
    id = $deliveryProfile.node.id;
  }
}

$profileDetails = Invoke-RestMethod -Method Post -Uri $uri -Headers $jsonHeaders -Body (ConvertTo-Json -Depth 3 $getDeliveryProfileData)
```

The list of existing delivery method definitions can be easily shown:

```powershell
$methodsToDelete = $profileDetails.data.deliveryProfile.profileLocationGroups.locationGroupZones.edges.node.methodDefinitions.edges.node.id
$methodsToDelete
```

Then follow the instructions in 'Read shipping rate data' and 'Uploading rates' up to the point where
`$profileLocationGroupUpdateInput` is created.

Then use both `$profileLocationGroupUpdateInput` and `$methodsToDelete` to update the rates.

```powershell
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

$updateRatesResult = Invoke-RestMethod -Method Post -Uri $uri -Headers $jsonHeaders -Body (ConvertTo-Json -Depth 15 $updateRates)
$updateRatesResult
```

#### Separate delete and add

If the data doesn't update in one go cleanly, e.g. you get multiple rate definitions, try deleting separately. Run this, then query the rates again (above, at the start of 'Replacing existing rates') and see if there are any more to delete.

You may need to delete and check multiple times, as it may only do a few (e.g. 40) rows at a time. Check the IDs change each time you query a batch to be deleted:

```powershell
$updateRates = @{
  query = $updateProfileQuery;
  variables = @{
    id = $deliveryProfile.node.id;
    profile = @{
      methodDefinitionsToDelete = $methodsToDelete
    }
  }
}

$updateRatesResult = Invoke-RestMethod -Method Post -Uri $uri -Headers $jsonHeaders -Body (ConvertTo-Json -Depth 15 $updateRates)
$updateRatesResult
```

New rates:

```powershell
$updateRates = @{
  query = $updateProfileQuery;
  variables = @{
    id = $deliveryProfile.node.id;
    profile = @{
      locationGroupsToUpdate = @( $profileLocationGroupUpdateInput )
    }
  }
}

$updateRatesResult = Invoke-RestMethod -Method Post -Uri $uri -Headers $jsonHeaders -Body (ConvertTo-Json -Depth 15 $updateRates)
$updateRatesResult
```


### Replacing the entire existing zones (not just rates, but the zone definitions)

Get details of the existing profile you want to replace:

```powershell
$getDeliveryProfileData = @{
  query = $getDeliveryProfileQuery;
  variables = @{
    id = $deliveryProfile.node.id;
  }
}

$profileDetails = Invoke-RestMethod -Method Post -Uri $uri -Headers $jsonHeaders -Body (ConvertTo-Json -Depth 3 $getDeliveryProfileData)
```

Get all existing location group zones:

```powershell
$zonesToDelete = $profileDetails.data.deliveryProfile.profileLocationGroups.locationGroupZones.edges.node.zone.id
$zonesToDelete
```

Then delete them:

```powershell
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


Assigning products to profiles
-------------------------------

Get a list of all products you want to assign, e.g. from one vendor.

```powershell
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

```powershell
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

```powershell
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


Multiple profiles
-----------------

To update the rates in a second profile.

Follow the process up to **Get existing shipping profile information**, where you have retrieved `$deliveryProfiles`. To see all the profile names you can query `$deliveryProfiles.data.deliveryProfiles.edges`.

**Note:** Use the name of the rate you want to change, e.g. 'Wholesale Shipping':

```powershell
$deliveryProfile2 = $deliveryProfiles.data.deliveryProfiles.edges | Where-Object { $_.node.name -eq 'Wholesale Shipping' }
$deliveryProfile2.node.profileLocationGroups | Measure-Object
$locationGroupId2 = $deliveryProfile2.node.profileLocationGroups[0].locationGroup.id
```

### Rates to be deleted

Get the existing profile details:

```powershell
$getDeliveryProfileData2 = @{
  query = $getDeliveryProfileQuery;
  variables = @{
    id = $deliveryProfile2.node.id;
  }
}

$profileDetails2 = Invoke-RestMethod -Method Post -Uri $uri -Headers $jsonHeaders -Body (ConvertTo-Json -Depth 3 $getDeliveryProfileData2)
```

Then convert that to the rates to be deleted:

```powershell
$methodsToDelete2 = $profileDetails2.data.deliveryProfile.profileLocationGroups.locationGroupZones.edges.node.methodDefinitions.edges.node.id
$methodsToDelete2
```

### Rates to add

Then follow the instructions in 'Read shipping rate data' and 'Uploading rates' up to the point where
`$profileLocationGroupUpdateInput` is created.

Get the created zone IDs.

```powershell
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

$getDeliveryProfileZonesData2 = @{
  query = $getDeliveryProfileZonesQuery;
  variables = @{
    id = $deliveryProfile2.node.id;
  }
}

$profileZones2 = Invoke-RestMethod -Method Post -Uri $uri -Headers $jsonHeaders -Body (ConvertTo-Json -Depth 3 $getDeliveryProfileZonesData2)
$zoneIdsAndNames2 = $profileZones2.data.deliveryProfile.profileLocationGroups[0].locationGroupZones.edges.node.zone
$zoneIdsAndNames2 | Measure-Object
```

Read the data file and use it to build the zone updates adding the method definitions.

**Note:** Use the file name of the rates being changed, e.g. 'data/auspost-rates-discounted-insured-express-to-4kg.csv'

```powershell
$zoneRateData2 = Import-Csv 'data/auspost-rates-discounted-insured-express-to-5kg.csv'
$zonesToUpdate2 = [System.Collections.ArrayList]@()
$currentZone = $null
$zoneRateData2 | ForEach-Object {
  $line = $_
  if ($line.zone -ne $currentZone) {
    $currentZone = $line.zone
    $zone = $zoneIdsAndNames2 | Where-Object { $_.name -eq $currentZone}
    if (-not $zone) { throw "Zone $($_.name) not found" }
    $zoneInput = @{ id = $zone.id; methodDefinitionsToCreate = [System.Collections.ArrayList]@() }
    $i = $zonesToUpdate2.Add($zoneInput)
  }
  $weightConditionsInput = [System.Collections.ArrayList]@()
  if ([decimal]$line.lessThanKg) {
    $i = $weightConditionsInput.Add(@{ criteria = @{ unit = 'GRAMS'; value = [decimal]$line.lessThanKg * 1000; }; operator = 'LESS_THAN_OR_EQUAL_TO' })
  }
  if ([decimal]$line.greaterThanKg) {
    $i = $weightConditionsInput.Add(@{ criteria = @{ unit = 'GRAMS'; value = [decimal]$line.greaterThanKg * 1000; }; operator = 'GREATER_THAN_OR_EQUAL_TO' })
  }
  $methodInput = @{ 
    active = $true;
    name = $line.method;
    rateDefinition = @{ price = @{ amount = [decimal]$line.rateAud; currencyCode = 'AUD' } };
    weightConditionsToCreate = $weightConditionsInput;
  }
  $i = $zoneInput.methodDefinitionsToCreate.Add($methodInput)
}
$zonesToUpdate2 | ConvertTo-Json -Depth 6
```

We can then build the quety to update the zones:

```powershell
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

$profileLocationGroupUpdateInput2 = @{ id = $locationGroupId2; zonesToUpdate = $zonesToUpdate2 }
```

### Send the update

Then use both `$profileLocationGroupUpdateInput2` and `$methodsToDelete2` to update the rates.

Build the delete query:

```powershell
$updateRates = @{
  query = $updateProfileQuery;
  variables = @{
    id = $deliveryProfile2.node.id;
    profile = @{
      methodDefinitionsToDelete = $methodsToDelete2
    }
  }
}

$updateRatesResult = Invoke-RestMethod -Method Post -Uri $uri -Headers $jsonHeaders -Body (ConvertTo-Json -Depth 11 $updateRates)
$updateRatesResult
```

And then the update:

```powershell
$updateRates2 = @{
  query = $updateProfileQuery;
  variables = @{
    id = $deliveryProfile2.node.id;
    profile = @{
      locationGroupsToUpdate = @( $profileLocationGroupUpdateInput2 )
    }
  }
}

$updateRatesResult = Invoke-RestMethod -Method Post -Uri $uri -Headers $jsonHeaders -Body (ConvertTo-Json -Depth 11 $updateRates2)
$updateRatesResult
```

Check the rates have updated in the UI.


New rate update process
-----------------------

### Update rate data files

* Get the new rates from the Australia Post website, https://auspost.com.au/

* Open `data/auspost-rates-data.ods` (OpenOffice, or should also be able to open in MS Office)

* Check that there are no changes affective `Zones and countries` and `Domestic zones`, i.e. the zones are the same (just the rates change). If the zones or zone structure has changed, then updates may be a lot more involved than just a rate update.

* Update the `Rate data` sheet, fill in the new base values (in blue); check other values calculate correctly.

* Check Rate definitions 1 and Rate definitions contain the list of rates you want, or create your own.

e.g. Rate definitions 1 has retail rates for both standard and express options, up to 1 kg, for domestic + all zones.
Rate definitions 2 is desgined for wholesale products, with only express rates, with insurance, and larger weights (up to 5kg), but with a discount applied.

* Export the output tables `Zone rates 1` and `Zone rates 2` to CSV.

* Commit to track in source control

### Preparing for update

* Open a PowerShell terminal

* Check credentials working as per `Configure a Private App to get access to the API` and `Accessing the GraphQL API`

```powershell
$password = '<Password>'
$shopName = '<shop name>'

$uri = "https://$shopName.myshopify.com/admin/api/2021-01/graphql.json"
$headers = @{ 
  'Content-Type' = 'application/graphql';
  'X-Shopify-Access-Token' = $password
}

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

* Get profile information, following `Get existing shipping profile information`

```powershell
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
$defaultProfileId
```

### Replace rates for a profile for shoping

* Then follow the details in `Replacing existing rates`, starting with selecting the profile (change the name for a different profile):

```powershell
$profileName = 'General Profile'
$zoneRateDataFile = 'data/auspost-rates-to-1-5kg.csv'

$deliveryProfile = $deliveryProfiles.data.deliveryProfiles.edges | Where-Object { $_.node.name -eq $profileName }
$deliveryProfile.node.profileLocationGroups | Measure-Object
$locationGroupId = $deliveryProfile.node.profileLocationGroups[0].locationGroup.id
$profileLocationGroupInput = @{ id = $locationGroupId; zonesToCreate = $zonesToCreate }
```

Then getting the profile and checking the list of existing method definitions to delete:

```powershell
$getDeliveryProfileData = @{
  query = $getDeliveryProfileQuery;
  variables = @{
    id = $deliveryProfile.node.id;
  }
}

$profileDetails = Invoke-RestMethod -Method Post -Uri $uri -Headers $jsonHeaders -Body (ConvertTo-Json -Depth 3 $getDeliveryProfileData)

$methodsToDelete = $profileDetails.data.deliveryProfile.profileLocationGroups.locationGroupZones.edges.node.methodDefinitions.edges.node.id
$methodsToDelete
```

* With the data prepared above, load the CSV data following `Read shipping rate data` 

First, get the zone IDs

```powershell
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

Then read the data file and build the new rates (change the data file to match the profile)

```powershell
$zoneRateData = Import-Csv $zoneRateDataFile
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

* And then 'Uploading rates' up to the point where `$profileLocationGroupUpdateInput` is created.

```powershell
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
```

* Then use both `$profileLocationGroupUpdateInput` and `$methodsToDelete` to update the rates.

```powershell
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
$updateRatesResult | ConvertTo-Json -Depth 9
```

* After loading, check the rates in Shopify Admin > Settings > Shipping and delivery > Manage (for the profile), then scroll down and look at the rates.


### Replace additional profiles

* Follow the steps above in `Replace rates for a profile for shoping`, but for a different profile and data file, e.g.:

```powershell
$profileName = 'Wholesale Shipping'
$zoneRateDataFile = 'data/auspost-rates-discounted-insured-express-to-4kg.csv'

$deliveryProfile = $deliveryProfiles.data.deliveryProfiles.edges | Where-Object { $_.node.name -eq $profileName }
$deliveryProfile.node.profileLocationGroups | Measure-Object
$locationGroupId = $deliveryProfile.node.profileLocationGroups[0].locationGroup.id
$profileLocationGroupInput = @{ id = $locationGroupId; zonesToCreate = $zonesToCreate }
```
