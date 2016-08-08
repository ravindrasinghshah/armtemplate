{  
   "$schema":"https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
   "contentVersion":"1.0.0.0",
  "parameters": {
    "sqlVMName": {
      "type": "string"
    },
    "location": {
      "type": "string"
    },
    "adminUsername": {
      "type": "string"
    },
    "adminPassword": {
      "type": "securestring"
    },
    "domainName": {
      "type": "string"
    },
    "sqlAOPrepareModulesURL": {
      "type": "string"
    },
    "sqlAOPrepareConfigurationFunction": {
      "type": "string"
    },
    "sqlAOEPName": {
      "type": "string"
    },
    "sqlServerServiceAccountUserName": {
      "type": "string"
    },
    "sqlServerServiceAccountPassword": {
      "type": "securestring"
    },
    "sharePath": {
      "type": "string"
    },
    "adPDCVMName": {
      "type": "string"
    },
    "sqlwVMName": {
      "type": "string"
    },
    "fswModulesURL": {
      "type": "string"
    },
    "fswConfigurationFunction": {
      "type": "string"
    },
    "autoPatchingEnable": {
      "type": "bool"
    },
    "autoPatchingDay": {
      "type": "string"
    },
    "autoPatchingStartHour": {
      "type": "string"
    },
    "numberOfDisks": {
      "type": "string"
    },
    "workloadType": {
      "type": "string"
    },
    "sqlAutobackupStorageAccountName": {
      "type": "string",
      "metadata": {
        "description": "SQL Server Auto Backup Storage Account Name"
      }
    },
    "sqlAutobackupEncryptionPassword": {
      "type": "string",
      "metadata": {
        "description": "SQL Server Auto Backup Encryption Password"
      }
    }
  },
   "variables":{  
      "Monday":"[mod(div(add(add(24,int(parameters('autoPatchingStartHour'))),2),24),7)]",
      "Tuesday":"[mod(div(add(add(48,int(parameters('autoPatchingStartHour'))),2),24),7)]",
      "Wednesday":"[mod(div(add(add(72,int(parameters('autoPatchingStartHour'))),2),24),7)]",
      "Thursday":"[mod(div(add(add(96,int(parameters('autoPatchingStartHour'))),2),24),7)]",
      "Friday":"[mod(div(add(add(120,int(parameters('autoPatchingStartHour'))),2),24),7)]",
      "Saturday":"[mod(div(add(add(144,int(parameters('autoPatchingStartHour'))),2),24),7)]",
      "Sunday":"[mod(div(add(add(168,int(parameters('autoPatchingStartHour'))),2),24),7)]",
      "Never":"8",
      "Everyday":"0",
      "1":"Monday",
      "2":"Tuesday",
      "3":"Wednesday",
      "4":"Thursday",
      "5":"Friday",
      "6":"Saturday",
      "7":"Sunday",
      "8":"Monday",
      "0":"Everyday"
   },
   "resources":[  
      {  
         "type":"Microsoft.Compute/virtualMachines/extensions",
         "name":"[concat(parameters('sqlwVMName'),'/CreateFileShareWitness')]",
         "apiVersion":"2015-06-15",
         "location":"[parameters('location')]",
         "properties":{  
            "publisher":"Microsoft.Powershell",
            "type":"DSC",
            "typeHandlerVersion":"2.19",
            "autoUpgradeMinorVersion":false,
            "settings":{  
               "modulesURL":"[parameters('fswModulesURL')]",
               "configurationFunction":"[parameters('fswConfigurationFunction')]",
               "properties":{  
                  "domainName":"[parameters('domainName')]",
                  "SharePath":"[parameters('sharePath')]",
                  "adminCreds":{  
                     "userName":"[parameters('adminUserName')]",
                     "password":"privateSettingsRef:adminPassword"
                  }
               }
            },
            "protectedSettings":{  
               "items":{  
                  "adminPassword":"[parameters('adminPassword')]"
               }
            }
         }
      },
      {  
         "apiVersion":"2015-06-15",
         "type":"Microsoft.Compute/virtualMachines/extensions",
         "name":"[concat(parameters('sqlVMName'),'0/SqlVmIaasExtension')]",
         "location":"[parameters('location')]",
         "properties":{  
            "type":"SqlIaaSAgent",
            "publisher":"Microsoft.SqlServer.Management",
            "typeHandlerVersion":"1.2",
            "autoUpgradeMinorVersion":"true",
            "settings":{  
               "AutoTelemetrySettings":{  
                  "Region":"[parameters('location')]"
               },
               "AutoPatchingSettings":{  
                  "PatchCategory":"WindowsMandatoryUpdates",
                  "Enable":"[parameters('autoPatchingEnable')]",
                  "DayOfWeek":"[parameters('autoPatchingDay')]",
                  "MaintenanceWindowStartingHour":"[int(parameters('autoPatchingStartHour'))]",
                  "MaintenanceWindowDuration":"60"
               },
               "AutoBackupSettings":{  
                  "Enable":false,
                  "RetentionPeriod":"30",
                  "EnableEncryption":false
               }
            }
         }
      },
      {  
         "apiVersion":"2015-06-15",
         "type":"Microsoft.Compute/virtualMachines/extensions",
         "name":"[concat(parameters('sqlVMName'),'1/SqlVmIaasExtension')]",
         "location":"[parameters('location')]",
         "properties":{  
            "type":"SqlIaaSAgent",
            "publisher":"Microsoft.SqlServer.Management",
            "typeHandlerVersion":"1.2",
            "autoUpgradeMinorVersion":"true",
          "settings": {
            "AutoTelemetrySettings": {
              "Region": "[parameters('location')]"
            },
            "AutoPatchingSettings": {
              "PatchCategory": "WindowsMandatoryUpdates",
              "Enable": "[parameters('autoPatchingEnable')]",
              "DayOfWeek": "[variables(string(variables(parameters('autoPatchingDay'))))]",
              "MaintenanceWindowStartingHour": "[mod(add(int(parameters('autoPatchingStartHour')),2),24)]",
              "MaintenanceWindowDuration": "60"
            },
            "AutoBackupSettings": {
              "Enable": true,
              "RetentionPeriod": "30",
              "EnableEncryption": false
            },
            "protectedSettings": {
              "StorageUrl": "[reference(resourceId('Microsoft.Storage/storageAccounts', parameters('sqlAutobackupStorageAccountName')), '2015-06-15').primaryEndpoints['blob']]",
              "StorageAccessKey": "[listKeys(resourceId('Microsoft.Storage/storageAccounts', parameters('sqlAutobackupStorageAccountName')), '2015-06-15').key1]",
              "Password": "[parameters('sqlAutobackupEncryptionPassword')]"
            }
          }
         }
      },
      {  
         "type":"Microsoft.Compute/virtualMachines/extensions",
         "name":"[concat(parameters('sqlVMName'),'0/sqlAOPrepare')]",
         "apiVersion":"2015-06-15",
         "location":"[parameters('location')]",
         "dependsOn":[  
            "[concat('Microsoft.Compute/virtualMachines/',parameters('sqlwVMName'),'/extensions/CreateFileShareWitness')]",
            "[concat('Microsoft.Compute/virtualMachines/',parameters('sqlVMName'),'0/extensions/SqlVmIaasExtension')]"
         ],
         "properties":{  
            "publisher":"Microsoft.Powershell",
            "type":"DSC",
            "typeHandlerVersion":"2.19",
            "autoUpgradeMinorVersion":false,
            "settings":{  
               "modulesURL":"[parameters('sqlAOPrepareModulesURL')]",
               "configurationFunction":"[parameters('sqlAOPrepareConfigurationFunction')]",
               "properties":{  
                  "domainName":"[parameters('domainName')]",
                  "sqlAlwaysOnEndpointName":"[parameters('sqlAOEPName')]",
                  "adminCreds":{  
                     "userName":"[parameters('adminUserName')]",
                     "password":"privateSettingsRef:AdminPassword"
                  },
                  "sqlServiceCreds":{  
                     "userName":"[parameters('sqlServerServiceAccountUserName')]",
                     "password":"privateSettingsRef:SqlServerServiceAccountPassword"
                  },
                  "NumberOfDisks":"[parameters('numberOfDisks')]",
                  "WorkloadType":"[parameters('workloadType')]"
               }
            },
            "protectedSettings":{  
               "items":{  
                  "adminPassword":"[parameters('adminPassword')]",
                  "sqlServerServiceAccountPassword":"[parameters('sqlServerServiceAccountPassword')]"
               }
            }
         }
      }
   ],
   "outputs":{  

   }
}
