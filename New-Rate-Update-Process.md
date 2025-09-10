New rate update process
-----------------------

### Update rate data files

* Get the new rates from the Australia Post website, https://auspost.com.au/

* Open `data/auspost-rates-data.ods` (OpenOffice, or should also be able to open in MS Office)

* Check that there are no changes affective `Zones and countries` and `Domestic zones`, i.e. the zones are the same (just the rates change). If the zones or zone structure has changed, then updates may be a lot more involved than just a rate update.

* Update the `Rate data` sheet, fill in the new base values (in blue); check other values calculate correctly.

* Check Rate definitions 1 and Rate definitions contain the list of rates you want, or create your own.

e.g. Rate definitions 1 has retail rates for both standard and express options, up to 1 kg, for domestic + all zones.
Rate definitions 2 is designed for wholesale products, with only express rates, and with insurance, but with a discount applied.

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

Prepare JSON headers for parametized queries:

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
            methodDefinitions (first:40) {
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

$getDeliveryProfileZonesQuery = 'query($id: ID!)
{
  deliveryProfile (id: $id) {
    profileLocationGroups {
      locationGroupZones (first: 20) {
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

$updateProfileQuery = 'mutation($id: ID!, $profile: DeliveryProfileInput!) {
  deliveryProfileUpdate (id: $id, profile: $profile)
  {
    profile {
      id
      name
      profileLocationGroups {
        locationGroupZones (first: 20) {
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

```

### Replace rates for a profile for shopping

* Then follow the details in `Replacing existing rates`, starting with selecting the profile (change the name for a different profile):

```powershell
$profileName = 'General Profile'
$zoneRateDataFile = 'data/auspost-rates-to-5kg.csv'

$deliveryProfile = $deliveryProfiles.data.deliveryProfiles.edges | Where-Object { $_.node.name -eq $profileName }
$deliveryProfile.node.profileLocationGroups | Measure-Object
$locationGroupId = $deliveryProfile.node.profileLocationGroups[0].locationGroup.id
$profileLocationGroupInput = @{ id = $locationGroupId; zonesToCreate = $zonesToCreate }
```

* With the data prepared above, load the CSV data following `Read shipping rate data` 

First, get the zone IDs

```powershell
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
$profileLocationGroupUpdateInput = @{ id = $locationGroupId; zonesToUpdate = $zonesToUpdate }
```

### Delete existing

You may need more than one query to delete all the existing rates. Run until all `$methodsToDelete` rates are cleared, then immediately load new `$profileLocationGroupUpdateInput`

* (a) Getting the profile and checking the list of existing method definitions to delete:

```powershell
$getDeliveryProfileData = @{
  query = $getDeliveryProfileQuery;
  variables = @{
    id = $deliveryProfile.node.id;
  }
}

$profileDetails = Invoke-RestMethod -Method Post -Uri $uri -Headers $jsonHeaders -Body (ConvertTo-Json -Depth 3 $getDeliveryProfileData)

$methodsToDelete = $profileDetails.data.deliveryProfile.profileLocationGroups.locationGroupZones.edges.node.methodDefinitions.edges.node.id
$methodsToDelete | Measure-Object
```

* (b) Limit to 250 maximum, and delete

```powershell
$methodsToDeleteBatch = $methodsToDelete[0..249]

$deleteRates = @{
  query = $updateProfileQuery;
  variables = @{
    id = $deliveryProfile.node.id;
    profile = @{
      methodDefinitionsToDelete = $methodsToDeleteBatch
    }
  }
}

$deleteRatesResult = Invoke-RestMethod -Method Post -Uri $uri -Headers $jsonHeaders -Body (ConvertTo-Json -Depth 11 $deleteRates)
$deleteRatesResult | ConvertTo-Json -Depth 9
```

* Repeat (a) fetching and (b) deleting, above, until complete.

* If you check the Shopify Web UI, it will say "No rates." for all zones.

* Then use both `$profileLocationGroupUpdateInput` to update the rates.

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

$updateRatesResult = Invoke-RestMethod -Method Post -Uri $uri -Headers $jsonHeaders -Body (ConvertTo-Json -Depth 11 $updateRates)
$updateRatesResult | ConvertTo-Json -Depth 9
```

* After loading, check the rates in Shopify Admin > Settings > Shipping and delivery > Manage (for the profile), then scroll down and look at the rates.


### Replace additional profiles

* Follow the steps above in `Replace rates for a profile for shopping`, but for a different profile and data file, e.g.:

```powershell
$profileName = 'Wholesale Shipping'
$zoneRateDataFile = 'data/auspost-rates-discounted-insured-express-to-5kg.csv'

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
