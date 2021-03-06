## KQL_Cheat_Sheet

### Defender ATP

#### TVM Vulnerabilites

###### (critical 1days) 
**let newVuln = DeviceTvmSoftwareVulnerabilitiesKB**  
| where VulnerabilitySeverityLevel == "Critical"  
| where LastModifiedTime >ago(1day); DeviceTvmSoftwareInventoryVulnerabilities   
| join newVuln on CveId  
| summarize dcount(DeviceId) by DeviceName, DeviceId, Timestamp=LastModifiedTime, SoftwareName, SoftwareVendor, SoftwareVersion, VulnerabilitySeverityLevel, CvssScore, IsExploitAvailable  
| project Timestamp, DeviceName, SoftwareName, SoftwareVendor, SoftwareVersion, VulnerabilitySeverityLevel, CvssScore, IsExploitAvailable, DeviceId

###### URL: https://github.com/jangeisbauer/AdvancedHunting/blob/master/Detected%20new%20Vuln%20in%20Enterprise   
  
**DeviceTvmSoftwareInventoryVulnerabilities**  
| project  DeviceName, SoftwareName, CveId, SoftwareVersion, VulnerabilitySeverityLevel   
| join (DeviceTvmSoftwareVulnerabilitiesKB  
| project AffectedSoftware, VulnerabilityDescription , CveId , CvssScore , IsExploitAvailable) on CveId   
| project CveId , SoftwareName , SoftwareVersion , VulnerabilityDescription , VulnerabilitySeverityLevel, IsExploitAvailable , CvssScore   
| distinct SoftwareName , SoftwareVersion, CveId, VulnerabilityDescription , VulnerabilitySeverityLevel, IsExploitAvailable    
| sort by SoftwareName asc , SoftwareVersion  
____  
#### UEFI SCAN  
###### URL: https://www.microsoft.com/security/blog/2020/06/17/uefi-scanner-brings-microsoft-defender-atp-protection-to-a-new-level/

**DeviceEvents**  
| where ActionType == "AntivirusDetection"  
| extend ParsedFields=parse_json(AdditionalFields)  
| extend ThreatName=tostring(ParsedFields.ThreatName)  
| where ThreatName contains_cs "UEFI"  
| project ThreatName=tostring(ParsedFields.ThreatName), FileName, SHA1, DeviceName, Timestamp  
| limit 100  

**DeviceAlertEvents**  
| where Title has "UEFI"  
| summarize Titles=makeset(Title) by DeviceName, DeviceId, bin(Timestamp, 1d)  
| limit 100  

_____

#### Controlled Folder Access  
**DeviceEvents**  
| where ActionType in ('ControlledFolderAccessViolationAudited','ControlledFolderAccessViolationBlocked')

#### Exploit Protection / Network Protection  
**DeviceEvents**  
| where ActionType startswith 'ExploitGuard' and ActionType !contains 'NetworkProtection'

#### Scripts after download  
**DeviceProcessEvents**  
| where InitiatingProcessFileName in ("iexplore.exe","chrome.exe","msedge.exe","firefox.exe")  
| where ProcessCommandLine contains "wscript.exe"  

____  
### Azure AD

#### Conditional Access
  
*Changes to a conditional access-policy and by who*

**AuditLogs**  
| where Category == "Policy"  
| project  ActivityDateTime, ActivityDisplayName, TargetResources[0].displayName, InitiatedBy.user.userPrincipalName  

**SigninLogs**  
| where ConditionalAccessStatus == "success"  
| project AppDisplayName, Identity, ConditionalAccessStatus    

### Sentinel

#### Unfamiliar Sign-in & Suspicious URL clicked (https://emptydc.com/2020/08/12/thoughts-on-identity/)

**SecurityAlert**  
| where AlertName has "Unfamiliar sign-in properties"  
| where TimeGenerated > ago(1d)  
| extend $account1 = tostring(parse_json(Entities)[0].["Name"])  
| join (  
**SecurityAlert**  
| where AlertName has "Suspicious URL clicked"  
| where TimeGenerated > ago(30d)  
| extend $account2 = iif(isnotempty(tostring(parse_json(Entities)[4].["Name"])), tostring(parse_json(Entities)[4].["Name"]), tostring(parse_json(Entities)[1].["Name"]))  
| project TimeGenerated, DisplayName, $account2  
) on $left.$account1 == $right.$account2  
