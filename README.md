## Performing a Deployment from Azure DevOps to Azure App Services when IPSecurity Restriction is configured on the SCM site

You may get a requirement wherein you need to configure IPSecurtiy restriction on both main and SCM sites and perform deployment using Azure DevOps. In this scenario, the deployment fails because you have no idea what IP to whitelist for Azure DevOps. Every time it uses a different IP, so it's completely dynamic.

As per Azure DevOps product team, “Our IP addresses are not documented and can change time to time.” Hence, we cannot even whitelist a specific range of IP addresses.

After a lot of research, trial and error tactic, I wrote a PowerShell script to achieve this requirement.

You need to create three tasks in the DevOps pipeline to perform the below each:
1.	Obtain the IP address of DevOps Hosted agent machine wherein the build/release process is taking place, and whitelist that IP address in the SCM site.
2.	Perform a Deployment.
3.	Delete the whitelisted IP address (this fakes the scenario as if you have done no IP restriction setting)

#### - Script to obtain the IP address and whitelist to the SCM site:
```
    $FunctionName = "{WebApps Name}"
    $ResourceGroupName = "{Resource Group Name}"

    $HostedAgentIPAddress = Invoke-RestMethod http://ipinfo.io/json | Select -exp ip
    $HostedAgentIPAddress = $HostedAgentIPAddress + "/32"

    $GetResource = Get-AzureRmResource -ResourceGroupName $ResourceGroupName -ResourceType Microsoft.Web/sites/config -ResourceName   $FunctionName/web -ApiVersion 2018-02-01

    $IpSecurity = $GetResource.Properties
    $IPSecure = $IpSecurity.scmIpSecurityRestrictions

    $restriction = @{}
    $restriction = @{
                  ipAddress = $HostedAgentIPAddress;
                  action = 'Allow';
                  name = 'AzureDevOpsIp';
                  priority = 400;
                  description = 'This is Azure DevOps IP address, configured for deployment purpose';
                }

    #$IpSecurity.scmIpSecurityRestrictions = @()     #uncomment this line if there is no IP restriction setting in prior, and comment when there is at least one IP Restriction configured

    $IpSecurity.scmIpSecurityRestrictions+= $restriction

    Set-AzureRmResource -ResourceGroupName $ResourceGroupName -ResourceType Microsoft.Web/sites/config -ResourceName $FunctionName/web -ApiVersion 2018-02-01 -PropertyObject $IpSecurity -Force

    $location = Get-Location
    $IPSecure | Out-File -FilePath $location\temp.txt -Force 
```
#### - Script to delete the previously whitelisted IP:
```
    $Key = ""
    $Value = ""
    $SplitArray = ""
    $count = 1
    $currentIpSecurity = @()
    $location = Get-Location

    $myObject = New-Object System.Object

    foreach($line in [System.IO.File]::ReadLines("$location\temp.txt"))
    {
        if($line -ne "")
        {   
              $SplitArray = $line.Split(':')
              $Key = $SplitArray[0].Trim()
              $Value = $SplitArray[1].Trim()
                     
              if($count -le 6)
              {
                  if($Key -eq 'ipAddress'){      
                  $myObject | Add-Member -type NoteProperty -name ipAddress -Value $Value
                  $count++
                  }

                  elseif($Key -eq 'action'){      
                  $myObject | Add-Member -type NoteProperty -name action -Value $Value
                  $count++
                  }

                  elseif($Key -eq 'tag'){      
                  $myObject | Add-Member -type NoteProperty -name tag -Value $Value
                  $count++
                  }

                  elseif($Key -eq 'name'){      
                  $myObject | Add-Member -type NoteProperty -name name -Value $Value
                  $count++
                  }
                  
                  elseif($Key -eq 'priority'){      
                  $myObject | Add-Member -type NoteProperty -name priority -Value $Value
                  $count++
                  }

                  elseif($Key -eq 'description'){      
                  $myObject | Add-Member -type NoteProperty -name description -Value $Value
                  $count++
                  }

              }
              else
              {
                  $count = 1
                  $currentIpSecurity += $myObject
                  $myObject = New-Object System.Object

                  if($Key -eq 'ipAddress'){      
                  $myObject | Add-Member -type NoteProperty -name ipAddress -Value $Value
                  $count++
                  }

              }
        }
    }
    $currentIpSecurity += $myObject
    
    $FunctionName = "{WebApps Name}”
    $ResourceGroupName = "{Resource Group Name}"
    
    $GetResource = Get-AzureRmResource -ResourceGroupName $ResourceGroupName -ResourceType Microsoft.Web/sites/config -ResourceName     $FunctionName/web -ApiVersion 2018-02-01
    
    $IpSecurity = $GetResource.Properties
    $IpSecurity.scmIpSecurityRestrictions = $currentIpSecurity

    Set-AzureRmResource -ResourceGroupName $ResourceGroupName -ResourceType Microsoft.Web/sites/config -ResourceName $FunctionName/web -ApiVersion 2018-02-01 -PropertyObject $IpSecurity -Force
```
#### Note : Please fill-up all the properties if you are adding a new IP restriction setting directly from the portal.
