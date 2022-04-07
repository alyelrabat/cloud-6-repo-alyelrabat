param location string = 'west europe'

/// APP_ LINUX ///

param vnet1 string = 'app_vnet'
param nsgapp string = 'nsg-app-http-https'
param nsg_app_rules_ssh string = 'nsg_apprules_ssh'
//param nic_app_vm string  = 'nic_linux'
param pip_app_linux string = 'pip_linux_web'
// param vm_linux string = 'LinuxVM'
param VM_SCALE_SET string = 'vmscalesetapp'

/// ADMIN_WINDOWS ///

param vnet2 string = 'man_vnet'
param nsgadmin string = 'nsg-Management-3389'
param nic_admin_vm string  = 'nic_Windows'
param pip_admin_windows string = 'pip_windows_admin'
param vm_windows string = 'WindowsVM'
param OSVersionWin string = '2019-Datacenter'

/// KEYVAULT,KEYS,ENCRYPT - PARAM ///

param DISKencryptionsetname string = 'DiskEncryption'

@description('Username for the Win Virtual Machine.')
param adminUsernameWin string

@description('Password for the Win Virtual Machine.')
@minLength(14) 
@secure()
param adminPasswordWin string 

var backupFabric = 'Azure'
// var protec_container_app_linux = 'iaasvmcontainer;iaasvmcontainerv2;${resourceGroup().name};${vm_linux}'
// var protec_Item_app_linux = 'vm;iaasvmcontainerv2;${resourceGroup().name};${vm_linux}'
var protec_container_admin_win = 'iaasvmcontainer;iaasvmcontainerv2;${resourceGroup().name};${vm_windows}'
var protec_Item_admin_windows = 'vm;iaasvmcontainerv2;${resourceGroup().name};${vm_windows}'

var script64 = loadFileAsBase64('./bootstrapscript/apache.sh') 
param store_name string = 'stvm${uniqueString(resourceGroup().id)}'
param utcValue string = utcNow()
param apachefile string = 'zscript.sh'

param pubkey string = 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC/VNMqEoUH2yLjwoDxXyHZFkssHUNqTKSHck4QZTYxqhhrpOgM3r6Y2lzymj5lDDpogERNqZz9RyFOFUClByRIXN95Sq/EvMa8nwGr+F8go9EFsMq1W5EFmChPWPyT3+y1gz3JcacocPZr6ru99LTTFU6DRuRJtraYmjJ7oEAD012mXYEnN4rXUTLXPQbJmxHjP+sS9ye71VjbwRt/QfRGbH1KuvVZ2MLhgLfxOdNMEmns0XgOj4nhLAlpOXYlOXs6eNiwNGNDaGfDIk8KjWqQPkqpxNEzIFz5F9rYCDs8eaKv1VijKjgLTdrrVfuMvXKgl7FjYmMRFO+BeXD5WSgl rsa-key-20220317'
param applicationgatewayname string = 'app-gateway-name'
param passssl string

/// RESOURCEGROUP ///

/// VMSS_VNET /// VMSS_SUBNET /// APP_GATEWAY_SUBNET /// NSG_APP_SSH /// APP_GATEWAY /// PIP_NATGATEWAY /// NAT_GATEWAY /// PEER_VNET_APP /// 

resource VMSS_VNET 'Microsoft.Network/virtualNetworks@2021-05-01' = {
  name: vnet1
  location: location
   properties:{
    addressSpace: {
      addressPrefixes: [
        '10.20.20.0/24'
      ]
    }
    enableDdosProtection: false
  }
}

resource VMSS_SUBNET 'Microsoft.Network/virtualNetworks/subnets@2021-05-01' = {
  name: 'backendsubnet'
  parent: VMSS_VNET
  properties: {
    addressPrefix: '10.20.20.128/25'
    privateEndpointNetworkPolicies:'Enabled'
    privateLinkServiceNetworkPolicies: 'Enabled'
    natGateway: {
      id: NAT_GATEWAY.id
    }
  }
}

resource APP_GATEWAY_SUBNET 'Microsoft.Network/virtualNetworks/subnets@2021-05-01' = {
  name: 'subnet_appgateway'
  parent: VMSS_VNET
  properties: {
    addressPrefix: '10.20.20.0/25'
    networkSecurityGroup:{
      id: NSG_APP_GATEWAY.id
    }
    privateEndpointNetworkPolicies: 'Enabled'
    privateLinkServiceNetworkPolicies: 'Enabled'
    
    //serviceEndpoints: [
      //{ 
        //service: 'Microsoft.KeyVault'
      }
     // {  
        //service: 'Microsoft.Storage'
      }
    //]
  //}        
//}

resource APP_GATEWAY 'Microsoft.Network/applicationGateways@2020-11-01' = {
  name: applicationgatewayname 
  location: location
  zones: [
    '1'
  ]
  properties: {
    sku: {
      name: 'Standard_v2'
      tier: 'Standard_v2'
      capacity: 2
    }
    sslCertificates: [
      {
        name: 'sslcert_name'
        properties: {
          data: loadFileAsBase64('./SSL_CERT/SSLCERT.pfx')
          password: passssl
        }
      }
    ]
    authenticationCertificates: []
    gatewayIPConfigurations: [
      {
        name: 'gatewayfrontendIP'
        properties: {
          subnet: {
            id: APP_GATEWAY_SUBNET.id
          }
        }
      }
    ]
    frontendIPConfigurations: [
      {
        name: 'front_appgateway_ip'
        properties: {
          privateIPAllocationMethod: 'Dynamic'
          publicIPAddress: {
            id:PIP_APP_GW.id
          }
        }
      }
    ]
    frontendPorts: [
      {
        name: 'front_appgateway443'
        properties: {
          port: 443
        }
      }
      {
        name: 'front_appgateway_http80'
        properties: {
          port: 80
        }
      }
    ]
    backendAddressPools: [
      {
        name: 'backend_appgateway_pools'
        properties: {
          backendAddresses: []
        }
      }
    ]
    backendHttpSettingsCollection: [
      {
        name: 'backend_appgateway_http_set'
        properties: {
          port: 80
          protocol: 'Http'
          cookieBasedAffinity: 'Disabled'
          connectionDraining: {
            enabled: false
            drainTimeoutInSec: 1
          }
          pickHostNameFromBackendAddress: false
          requestTimeout: 30
        }
      }
    ]
    httpListeners: [
      {
        name: 'http_listners_appgateway'
        properties: {
          frontendIPConfiguration: {
            id: resourceId('Microsoft.Network/applicationGateways/frontendIPConfigurations/',applicationgatewayname , 'front_appgateway_ip')
          }
          frontendPort: {
            id: resourceId('Microsoft.Network/applicationGateways/frontendPorts/',applicationgatewayname , 'front_appgateway443')
          }
          protocol: 'Https'
          sslCertificate: {
            id: resourceId('Microsoft.Network/applicationGateways/sslCertificates/',applicationgatewayname , 'sslcert_name') /// ${applicationgatewayname}
          }
          requireServerNameIndication: false
          hostNames:[]
        }
      }
      {
        name: 'listner'
        properties: {
          frontendIPConfiguration: {
            id: resourceId('Microsoft.Network/applicationGateways/frontendIPConfigurations/',applicationgatewayname , 'front_appgateway_ip')
          }
          frontendPort: {
            id: resourceId('Microsoft.Network/applicationGateways/frontendPorts/',applicationgatewayname , 'front_appgateway_http80')
          }
          protocol: 'Http'
          requireServerNameIndication: false
          hostNames:[]
        }
      }
    ]
    urlPathMaps: []
    requestRoutingRules: [
      {
        name: 'backend_route_rule'
        properties: {
          ruleType: 'Basic'
          httpListener: {
            id: resourceId('Microsoft.Network/applicationGateways/httpListeners/',applicationgatewayname , 'http_listners_appgateway')
          }
          backendAddressPool: {
            id: resourceId('Microsoft.Network/applicationGateways/backendAddressPools/',applicationgatewayname , 'backend_appgateway_pools')
          }
          backendHttpSettings: {
            id: resourceId('Microsoft.Network/applicationGateways/backendHttpSettingsCollection/',applicationgatewayname , 'backend_appgateway_http_set')
          }
        }
      }
      {
        name: 'redirect_route_rule'
        properties: {
          ruleType: 'Basic'
          httpListener: {
            id: resourceId('Microsoft.Network/applicationGateways/httpListeners/',applicationgatewayname , 'listner')
          }
          redirectConfiguration: {
            id: resourceId('Microsoft.Network/applicationGateways/redirectConfigurations/',applicationgatewayname , 'http_red_https')
          }
        }
      }
    ]
    privateLinkConfigurations:[]
    rewriteRuleSets:[]
    redirectConfigurations: [
      {
        name: 'http_red_https'
        properties: {
          redirectType: 'Permanent'
          targetListener: {
            id: resourceId('Microsoft.Network/applicationGateways/httpListeners/',applicationgatewayname , 'http_listners_appgateway')
          }
          includePath: true
          includeQueryString: true
          requestRoutingRules: [
            {
              id: resourceId('Microsoft.Network/applicationGateways/requestRoutingRules/',applicationgatewayname , 'redirect_route_rule')
            }
          ]
        }
      }
    ]
  }
}

resource NSG_APP_SSH 'Microsoft.Network/networkSecurityGroups/securityRules@2021-05-01' = { 
  parent: NSG_APP_GATEWAY
  name: nsg_app_rules_ssh
  properties: {
    protocol: 'Tcp'
    sourcePortRange: '*'
    destinationPortRange: '22'
    sourceAddressPrefix: NIC_ADMIN_WIN.properties.ipConfigurations[0].properties.privateIPAddress
    destinationAddressPrefix: '*'//NIC_APP_LINUX.properties.ipConfigurations[0].properties.privateIPAddress
    access: 'Allow'
    priority: 140
    direction: 'Inbound'
  }
}

resource NSG_APP_GATEWAY 'Microsoft.Network/networkSecurityGroups@2021-05-01' = {
  name: nsgapp
  location: location
  properties: {
    securityRules: [
      {
        name: 'app_nsg_https'
        properties: {
          protocol: 'Tcp'
          sourcePortRange: '*'
          destinationPortRange: '443'
          sourceAddressPrefix: '*'
          destinationAddressPrefix: '*'
          access: 'Allow'
          priority: 100
          direction: 'Inbound'
          sourcePortRanges: []
          destinationPortRanges: []
          sourceAddressPrefixes: []
          destinationAddressPrefixes: []
        } 
      }
      {
        name: 'app_nsg_http'
        properties: {
          protocol: 'Tcp'
          sourcePortRange: '*'
          destinationPortRange: '80'
          sourceAddressPrefix: '*'
          destinationAddressPrefix: '*'
          access: 'Allow'
          priority: 120
          direction: 'Inbound'
          sourcePortRanges: []
          destinationPortRanges: []
          sourceAddressPrefixes: []
          destinationAddressPrefixes: []
        }
      }
      {
        name: 'appgateway_in'
        properties: {
          protocol: 'Tcp'
          sourcePortRange: '*'
          destinationPortRange: '65200-65535'
          sourceAddressPrefix: '*'
          destinationAddressPrefix: '*'
          access: 'Allow'
          priority: 150
          direction: 'Inbound'
          sourcePortRanges: []
          destinationPortRanges: []
          sourceAddressPrefixes: []
          destinationAddressPrefixes: []
        }
      }
      {
        name: 'port_8080'
        properties: {
          protocol: 'Tcp'
          sourcePortRange: '*'
          destinationPortRange: '80'
          sourceAddressPrefix: '*'
          destinationAddressPrefix: '*'
          access: 'Allow'
          priority: 160
          direction: 'Inbound'
          sourcePortRanges: []
          destinationPortRanges: []
          sourceAddressPrefixes: []
          destinationAddressPrefixes: []
        }
      }
      {
        name: 'port_443'
        properties: {
          protocol:'Tcp'
          sourcePortRange: '*'
          destinationPortRange:'443'
          sourceAddressPrefix:'*'
          destinationAddressPrefix:'*'
          access: 'Allow'
          priority: 180
          direction: 'Outbound'
          sourcePortRanges: []
          destinationPortRanges: []
          sourceAddressPrefixes: []
          destinationAddressPrefixes: []
        }
      }
    ]
  }
}

resource PIP_NATGATEWAY'Microsoft.Network/publicIPAddresses@2020-11-01' = {
  name: 'pipoutbound'
  location: location
  sku: {
    tier: 'Regional'
    name: 'Standard'
  }
  zones: [
    '1'
  ]
  properties: {
    publicIPAllocationMethod: 'Static'
  }
}

resource NAT_GATEWAY 'Microsoft.Network/natGateways@2021-05-01' = {
  name: 'natgateway'
  location: location
  sku: {
    name: 'Standard'
  }
  properties: {
    idleTimeoutInMinutes: 4
    publicIpAddresses: [
      {
        id: PIP_NATGATEWAY.id
      }
    ]
  }
}

resource PEER_VNET_APP 'Microsoft.Network/virtualNetworks/virtualNetworkPeerings@2020-05-01' = {
  parent: VMSS_VNET
  name: '${vnet1}-${vnet2}'
  properties: {
    allowVirtualNetworkAccess: true
    allowForwardedTraffic: false
    allowGatewayTransit: false
    useRemoteGateways: false
    remoteVirtualNetwork: {
      id:ADMIN_VNET.id
    }
  }
  dependsOn: [
    ADMIN_SUBNET
    VMSS_SUBNET
  ]
}

/// NIC_APP_LINUX /// PIP_APP_GW /// VMSS_APP_LINUX /// NSG_VMSS /// AUTOSCALE_VMSS ///

//resource NIC_APP_LINUX 'Microsoft.Network/networkInterfaces@2020-06-01' = {
  //name: nic_app_vm
  //location: location
  //properties: {
    //ipConfigurations: [
      //{
        //name: 'ipconfig1'
        //properties: {
          //privateIPAddress:'10.10.10.8'
          //privateIPAllocationMethod: 'Dynamic'
          //subnet: {
            //id: VMSS_SUBNET.id
          //}
          //primary: true
          //privateIPAddressVersion: 'IPv4'
        //}
      //}
    //] 
    //dnsSettings: {
     //dnsServers: []
    //}
    //enableAcceleratedNetworking: false
    //enableIPForwarding: false
  //}
//}

resource PIP_APP_GW 'Microsoft.Network/publicIPAddresses@2020-06-01' = {
  name: pip_app_linux
  location: location
  sku: {
    name: 'Standard'
  }
  zones: [
    '1'
  ]
  properties: {
    publicIPAllocationMethod: 'Static'
  }
}

resource VMSS_APP_LINUX 'Microsoft.Compute/virtualMachineScaleSets@2021-11-01' = {
  name: VM_SCALE_SET
  location: location
  sku: {
    name: 'Standard_B1s'
    tier: 'Standard'
    capacity: 1
  }
  properties: {
    singlePlacementGroup: false
    orchestrationMode: 'Flexible'
    virtualMachineProfile: {
      osProfile: {
        customData: script64
        computerNamePrefix: VM_SCALE_SET
        adminUsername: '${VM_SCALE_SET}user'
        adminPassword: null
        linuxConfiguration: {
          disablePasswordAuthentication: true
        ssh: {
          publicKeys: [
            {
              keyData: pubkey
              path: '/home/${VM_SCALE_SET}user/.ssh/authorized_keys'
            }
          ]
        }
          provisionVMAgent: true
          patchSettings: {
            patchMode: 'ImageDefault'
            assessmentMode: 'ImageDefault'
          }
        }
        secrets: []
        allowExtensionOperations: true
      }
      storageProfile: {
        osDisk: {
          osType: 'Linux'
          createOption: 'FromImage'
          caching: 'ReadWrite'
          managedDisk: {
            storageAccountType: 'Premium_LRS'
            diskEncryptionSet: {
              id:DISK_ENCRYPT_SET.id
            }
          }
          diskSizeGB: 30
        }
        imageReference: {
          publisher: 'Canonical'
          offer: 'UbuntuServer'
          sku: '18.04-LTS'
          version: 'latest'
        }
      }
      extensionProfile: {
        extensions: [
          {
            name: 'HealthExtension'
            properties: {
              autoUpgradeMinorVersion: false
              publisher: 'Microsoft.ManagedServices'
              type: 'ApplicationHealthLinux'
              typeHandlerVersion: '1.0'
              settings: {
                protocol: 'http'
                port: 80
                requestPath: '/'
              }
            }
          }
        ]
      }
      networkProfile: {
        networkApiVersion:'2020-11-01'
        networkInterfaceConfigurations: [
          {
            name: '${VM_SCALE_SET}nic'
            properties: {
              primary: true
              enableAcceleratedNetworking: false
              enableIPForwarding: false
              ipConfigurations: [
                {
                  name: '${VM_SCALE_SET}ipconfig'
                  properties: {
                    subnet: {
                      id: VMSS_SUBNET.id
                    }
                    privateIPAddressVersion: 'IPv4'
                    applicationGatewayBackendAddressPools: [
                      {
                        id: APP_GATEWAY.properties.backendAddressPools[0].id
                      }
                    ]
                  }
                }
              ]
              networkSecurityGroup: {
                id:NSG_APP_GATEWAY.id
            }
           }
          }
        ]
      }
    }
    platformFaultDomainCount: 1
    automaticRepairsPolicy: {
      enabled: true
      gracePeriod: 'PT30M'
    }
  }
}

resource NSG_VMSS 'Microsoft.Network/networkSecurityGroups@2020-11-01' = {
  name: 'nsg_vmss'
  location: location
  properties: {
    securityRules: []
  }
}

resource AUTOSCALE_VMSS 'microsoft.insights/autoscalesettings@2021-05-01-preview' = {
  name: 'autoscale_settings_vmss'
  location: location
  properties: {
    profiles: [
      {
        name: 'Profile1'
        capacity: {
          minimum: '1'
          maximum: '3'
          default: '1'
        }
        rules: [
          {
            metricTrigger: {
              metricName: 'Percentage CPU'
              metricResourceUri: VMSS_APP_LINUX.id
              timeGrain: 'PT1M'
              statistic: 'Average'
              timeWindow: 'PT10M'
              timeAggregation: 'Average'
              operator: 'GreaterThan'
              threshold: 75
              dividePerInstance: false
            }
            scaleAction: {
              direction: 'Increase'
              type: 'ChangeCount'
              value: '1'
              cooldown: 'PT1M'
            }
          }
          {
            metricTrigger: {
              metricName: 'Percentage CPU'
              metricResourceUri: VMSS_APP_LINUX.id
              timeGrain: 'PT1M'
              statistic: 'Average'
              timeWindow: 'PT5M'
              timeAggregation: 'Average'
              operator: 'LessThan'
              threshold: 25
              dividePerInstance: false
            }
            scaleAction: {
              direction: 'Decrease'
              type: 'ChangeCount'
              value: '1'
              cooldown: 'PT1M'
            }
          }
        ]
      }
    ]
    enabled: true
    targetResourceUri: VMSS_APP_LINUX.id
  }
}

/// VNET_ADMIN /// SUBNET_ADMIN /// NSG_ADMIN /// PEER_VNET_ADMIN /// 

resource ADMIN_VNET 'Microsoft.Network/virtualNetworks@2021-05-01' = {
  name: vnet2
  location: location
   properties:{
    addressSpace: {
      addressPrefixes: [
        '10.10.10.0/24'
      ]
    }
    enableDdosProtection: false
  }
}

resource ADMIN_SUBNET 'Microsoft.Network/virtualNetworks/subnets@2021-05-01' = {
  name: '${vnet2}adminserver_subnet'
  parent: ADMIN_VNET
  
  properties: {
    addressPrefix: '10.10.10.0/25'
    delegations:[]
    privateEndpointNetworkPolicies: 'Enabled'
    privateLinkServiceNetworkPolicies: 'Enabled'
    networkSecurityGroup:{
      id: NSG_ADMIN.id
    }
    serviceEndpoints: [
      {
        service: 'Microsoft.KeyVault'
        locations: [
          '*'
        ]
      }
    ]
  }
}

resource NSG_ADMIN 'Microsoft.Network/networkSecurityGroups@2021-05-01' = {
  name: nsgadmin
  location: location
  properties: {
    securityRules: [
      {
        name: 'admin_nsg'
        properties: {
          protocol: 'Tcp'
          sourcePortRange: '*'
          destinationPortRange: '3389'
          sourceAddressPrefix: '*'
          destinationAddressPrefix: '*'
          access: 'Allow'
          priority: 160
          direction: 'Inbound'
          sourcePortRanges: []
          destinationPortRanges: []
          sourceAddressPrefixes: []
          destinationAddressPrefixes: []
        }
      }
    ]
  }
}

resource PEER_VNET_ADMIN'Microsoft.Network/virtualNetworks/virtualNetworkPeerings@2020-05-01' = {
  parent: ADMIN_VNET
  name: '${vnet2}-${vnet1}'
  properties: {
    allowVirtualNetworkAccess: true
    allowForwardedTraffic: false
    allowGatewayTransit: false
    useRemoteGateways: false
    remoteVirtualNetwork: {
      id:VMSS_VNET.id
    }
  }
  dependsOn: [
    ADMIN_SUBNET
    VMSS_SUBNET
  ]
}

/// NIC_ADMIN_WIN /// PIP_ADMIN /// VM_ADMIN_WIN ///

resource NIC_ADMIN_WIN 'Microsoft.Network/networkInterfaces@2020-11-01' = {
  name: nic_admin_vm
  location: location
  properties: {
    ipConfigurations: [
      {
        name: 'ipconfig1'
        properties: {
          privateIPAddress:'10.10.10.16'
          privateIPAllocationMethod: 'Dynamic'
          publicIPAddress: {
            id: PIP_ADMIN.id
          }
          subnet: {
            id: ADMIN_SUBNET.id
          }
          primary: true
          privateIPAddressVersion: 'IPv4'
        }
      }
    ] 
    dnsSettings: {
      dnsServers: []
    }
    enableAcceleratedNetworking: false
    enableIPForwarding: false
  }
}

resource PIP_ADMIN  'Microsoft.Network/publicIPAddresses@2020-06-01' = {
  name: pip_admin_windows
  location: location
  sku: {
   name: 'Standard'
  }
  zones: [
    '1'
  ]
  properties: {
    publicIPAllocationMethod: 'Static'
  }
}

resource VM_ADMIN_WIN 'Microsoft.Compute/virtualMachines@2021-03-01' = {
  name: vm_windows
  location: location
  zones: [
    '1'
  ]
  properties: {
    hardwareProfile: {
      vmSize: 'Standard_B1s'
    }
    osProfile: {
      computerName: vm_windows
      adminUsername: adminUsernameWin
      adminPassword: adminPasswordWin
      windowsConfiguration: {
        enableAutomaticUpdates: true
        provisionVMAgent: true
        patchSettings: {
          patchMode: 'AutomaticByOS'
          assessmentMode: 'ImageDefault'
          enableHotpatching: false
        }
      }
      secrets: []
      allowExtensionOperations: true
    }
    storageProfile: {
      imageReference: {
        publisher: 'MicrosoftWindowsServer'
        offer: 'WindowsServer'
        sku: OSVersionWin
        version: 'latest'
      }
      osDisk: {
        osType: 'Windows'
        createOption: 'FromImage'
        caching: 'ReadWrite'
        managedDisk: {
          diskEncryptionSet: {
            id: DISK_ENCRYPT_SET.id
          }
          storageAccountType: 'StandardSSD_LRS'
        }
        deleteOption: 'Delete'
      }
      dataDisks: []
    }
    networkProfile: {
      networkInterfaces: [
        {
          id: NIC_ADMIN_WIN.id
          properties: {
            deleteOption: 'Detach'
          }
        }
      ]
    }
    diagnosticsProfile: {
      bootDiagnostics: {
        enabled: true
      }
    }
  }
}

/// DISK_ENCRYPT_SET /// KEY_VAULT /// MAN_ID /// ACCES_POL /// KEY ///

resource DISK_ENCRYPT_SET 'Microsoft.Compute/diskEncryptionSets@2021-08-01' = {  
  name: DISKencryptionsetname
  location:location
  identity: {
    type:'SystemAssigned'
  }
  properties: {
    rotationToLatestKeyVersionEnabled: true
    activeKey: {
      sourceVault: {
        id: KEY_VAULT.id
      }
      keyUrl: KEY.properties.keyUriWithVersion
    }
    encryptionType: 'EncryptionAtRestWithCustomerKey'
  }
}

resource KEY_VAULT 'Microsoft.KeyVault/vaults@2021-06-01-preview' = {
  name: 'kluisproject46'
  location: location
  properties: {
    networkAcls: {
      defaultAction: 'Allow'
      bypass: 'AzureServices'
    }
    enabledForDeployment: true
    enabledForTemplateDeployment: true
    enabledForDiskEncryption: true
    enableSoftDelete:true
    enablePurgeProtection: true
    softDeleteRetentionInDays: 7
    enableRbacAuthorization: false
    publicNetworkAccess: 'Enabled'
    tenantId: 'de60b253-74bd-4365-b598-b9e55a2b208d'
    accessPolicies: [
      {
        tenantId: 'de60b253-74bd-4365-b598-b9e55a2b208d'
        objectId: '9346c2ae-a148-4d97-b928-1aa7a575c37e'
        permissions: {
          keys: [
            'backup'
            'create'
            'decrypt'
            'delete'
            'encrypt'
            'purge'
            'get'
            'import'
            'list'
            'recover'
            'release'
            'restore'
            'getrotationpolicy'
            'setrotationpolicy'
            'rotate'
            'sign'
            'unwrapKey'
            'update'
            'verify'
            'wrapKey'
          
          ]
           secrets: [
            'list'
            'set'
            'get'
            'delete'
            'backup'
            'restore'
            'recover'

          ]
          certificates: [
            'get'
            'backup'
            'create'
            'delete'
            'recover'
            'restore'
            'list'
            'managecontacts'
            'manageissuers'
            'import'
            'update'
            'listissuers'
            'getissuers'
            'setissuers'
            'deleteissuers'
          ]
        }
      }
    ]
    sku: {
      name: 'standard'
      family: 'A'
    }
  }
}

resource MAN_ID 'Microsoft.ManagedIdentity/userAssignedIdentities@2018-11-30' = {
  name: 'ManID'
  location: location
  dependsOn: [
    KEY_VAULT
  ]
}

resource ACCES_POL 'Microsoft.KeyVault/vaults/accessPolicies@2021-10-01' = { 
  name:  'add'
  parent: KEY_VAULT
  properties: {
    accessPolicies:[
      {
        tenantId: 'de60b253-74bd-4365-b598-b9e55a2b208d'
        objectId: DISK_ENCRYPT_SET.identity.principalId
        permissions: {
          keys: [
            'get'
            'wrapKey'
            'unwrapKey'
            'encrypt'
            'decrypt'

          ]
          secrets: []
          certificates: []
        }
        
      }
      {
        tenantId: 'de60b253-74bd-4365-b598-b9e55a2b208d'
        objectId: MAN_ID.properties.principalId
        permissions: {
          keys: [
            'get'
            'list'
            'unwrapKey'
            'wrapKey'
          ]
          secrets: []
          certificates: []
        }
      }
    ]
  }
}

resource KEY 'Microsoft.KeyVault/vaults/keys@2021-11-01-preview' = {
  name: 'RSAkey'
  parent: KEY_VAULT
  properties: {
    keyOps: [
      'decrypt'
      'encrypt'
      'unwrapKey'
      'wrapKey'
      'sign'
      'verify'
    ]
    attributes: {
      enabled: true
    }
    keySize: 4096
    kty:'RSA'
  }
}

resource SECRET 'Microsoft.KeyVault/vaults/secrets@2021-10-01' = {
  parent: KEY_VAULT
  name: 'ssh'
  properties: {
    value: loadTextContent('./keysecret/secretkeyproject.pub')
  }
}

/// STOR_ACC /// BLOB /// BLOB_CON /// REC_VAULT /// BACK_POL /// APP_PROTEC /// ADMIN_PROTEC ///

resource STOR_ACC 'Microsoft.Storage/storageAccounts@2021-08-01' = {
  name: 'storageproject1'
  location: location
  kind: 'StorageV2'
  identity: {
    type: 'UserAssigned'
    userAssignedIdentities: {
      '${MAN_ID.id}': {}
    }
  }
  sku:{
    name: 'Standard_ZRS'
  } 
  properties: {
    networkAcls: {
      bypass: 'AzureServices'
      virtualNetworkRules: []
      ipRules: []
      defaultAction: 'Allow'
    }

    defaultToOAuthAuthentication: false
    allowCrossTenantReplication: false
    allowBlobPublicAccess: false
    accessTier: 'Hot'
    supportsHttpsTrafficOnly: true
    minimumTlsVersion: 'TLS1_2'
    largeFileSharesState: 'Disabled'
    allowSharedKeyAccess: true
    encryption: {
      identity: {
        userAssignedIdentity: MAN_ID.id
      }
      keySource: 'Microsoft.Keyvault'
      keyvaultproperties: {
         keyname: 'RSAkey'    
         keyvaulturi:KEY_VAULT.properties.vaultUri
      }
      services: {
        blob: {
          enabled: true
          keyType: 'Account'
        }
        file: {
          keyType: 'Account'
          enabled: true
        }
        queue: {
          enabled: true
          keyType: 'Account'
        }
        table: {
          keyType: 'Account'
          enabled: true
        }
      }
    }
  }
}

resource BLOB 'Microsoft.Storage/storageAccounts/blobServices@2021-08-01' = {
  name: 'default'
  parent:  STOR_ACC
  properties: {
    containerDeleteRetentionPolicy: {
      days: 30
      enabled: true
    }
    changeFeed: {
      enabled: false
    }
    automaticSnapshotPolicyEnabled: true
    isVersioningEnabled: true
    restorePolicy: {
      enabled: true
      days: 7
    }

  }
}
   
resource BLOB_CON 'Microsoft.Storage/storageAccounts/blobServices/containers@2021-08-01'={
  parent: BLOB
  name: 'script64'
  properties: {
    publicAccess: 'None'
  }
}

resource REC_VAULT 'Microsoft.RecoveryServices/vaults@2021-08-01' = { 
  name: 'recoveryvault'
  location: location
  sku: {
    tier: 'Standard'
    name: 'RS0'
  }
  properties: {}
} 

resource BACK_POL 'Microsoft.RecoveryServices/vaults/backupPolicies@2021-12-01' = {
 location: location
 parent: REC_VAULT
 name: 'back_policy'
 properties: {
   backupManagementType: 'AzureIaasVM'
   schedulePolicy: {
     schedulePolicyType: 'SimpleSchedulePolicy'
     scheduleRunFrequency: 'Daily'
     scheduleRunTimes: [
       '2022-03-03T04:00:00Z'
     ]
     scheduleWeeklyFrequency: 0
    }
    retentionPolicy: {
      retentionPolicyType: 'LongTermRetentionPolicy'
      dailySchedule: {
        retentionTimes: [
          '2022-03-03T04:00:00Z'
        ]
        retentionDuration: {
          count: 7
          durationType: 'Days'
        }
      }
    }
    instantRpRetentionRangeInDays: 2
    timeZone: 'W. Europe Standard Time'
  }
}

resource PROTEC_VM_ADMIN 'Microsoft.RecoveryServices/vaults/backupFabrics/protectionContainers/protectedItems@2021-12-01' = {
  name: '${'recoveryvault'}/${backupFabric}/${protec_container_admin_win}/${protec_Item_admin_windows}' 
  properties: {
    protectedItemType: 'Microsoft.Compute/virtualMachines'
    policyId: BACK_POL.id
    sourceResourceId: VM_ADMIN_WIN.id
  }
} 
 
resource DEPLOY_SCRIPT 'Microsoft.Resources/deploymentScripts@2020-10-01' = {
  name: 'deployscript-upload-blob-${utcValue}'
  location: location
  kind: 'AzureCLI'
  properties: {
    azCliVersion: '2.26.1'
    timeout: 'PT5M'
    retentionInterval: 'PT1H'
    environmentVariables: [
      {
        name: 'AZURE_STORAGE_ACCOUNT'
        value: store_name
      }
      {
        name: 'AZURE_STORAGE_KEY'
        secureValue: STOR_ACC.listKeys().keys[0].value
      }
      {
        name: 'CONTENT'
        value: loadFileAsBase64('./bootstrapscript/apache.sh')
      }
    ]
    
    scriptContent: 'echo $CONTENT | base64 -d > ${apachefile} && az storage blob upload -f ${apachefile} -c ${script64} -n ${apachefile}'
  }
}




