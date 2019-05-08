{
    "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "metadata": {
      "comments": "This template is released under an as-is, best effort, and is community supported.",
      "author": "Matt McLimans (mmclimans@paloaltonetworks.com)"
    },
    "parameters": {
      "licenseType": {
        "type": "string",
        "defaultValue": "byol",
        "allowedValues": [
          "byol",
          "bundle1",
          "bundle2",
          "bundle2-for-ms"
        ],
        "metadata": {
          "description": "More info: https://docs.paloaltonetworks.com/vm-series/8-1/vm-series-deployment/license-the-vm-series-firewall/license-typesvm-series-firewalls"
        }
      },
      "(opt.) BootstrapStorageAccount": {
        "type": "string",
        "defaultValue": "",
        "metadata": {
          "description": "(Optional) To bootstrap the firewalls, enter the Azure storage account name that contains the bootstrap file share."
        }
      },
      "(opt.) BootstrapAccessKey": {
        "type": "securestring",
        "defaultValue": "",
        "metadata": {
          "description": "(Optional) To bootstrap the firewalls, enter the Azure storage account access key."
        }
      },
      "(opt.) BootstrapFileShareName": {
        "type": "string",
        "defaultValue": "",
        "metadata": {
          "description": "(Optional) To bootstrap the firewalls, enter the name of the Azure storage account file share that contains the bootstrap configuration."
        }
      },
      "(opt.) BootstrapShareDirectory": {
        "defaultValue": "",
        "type": "String",
        "metadata": {
          "description": "(Optional) To bootstrap from multiple bootstrap directories within the same file share, enter the subdirectory hosting the bootstrap files."
        }
      }
    },
    "variables": {
      "COMMENT_global": "GLOBAL VARIABLES SHARED AMONG DEPLOYED RESOURCES",
      "global_var_resource_group": "[resourceGroup().name]",
      "global_var_idleTimeoutInMinutes": 4,
      "global_var_allocationMethod": "Static",
      "global_var_networkVersion": "IPv4",
      "global_var_apiVersion": "2018-06-01",
      "global_var_tier": "Regional",
      "global_var_sku": "Standard",
      "global_vnet_name": "vmseries-vnet",
      "global_vnet_resource_group": "[resourceGroup().name]",
      "global_vnet_option": "Create new VNET",
      "global_vnet_subnet0_name": "mgmt-subnet",
      "global_vnet_subnet1_name": "untrust-subnet",
      "global_vnet_subnet2_name": "trust-subnet",
      "global_vnet_subnet3_name": "lb-subnet",
      "global_fw_mgmtnsg_name": "vmseries-nsg-mgmt",
      "global_fw_dataNsg_name": "vmseries-nsg-data",
      "global_fw_interface0_pip_option": "Yes",
      "global_fw_interface1_pip_option": "Yes",
      "global_fw_storageAccountType": "Standard_LRS",
      "global_fw_adminUsername": "paloalto",
      "global_fw_adminPassword": "PanPassword123!",
      "global_fw_diskSizeGB": 60,
      "global_fw_bootstrap": "[concat('storage-account=', parameters('(opt.) BootstrapStorageAccount'), ',access-key=', parameters('(opt.) BootstrapAccessKey'), ',file-share=', parameters('(opt.) BootstrapFileShareName'), ',share-directory=', parameters('(opt.) BootstrapShareDirectory'))]",
      "global_fw_publisher": "paloaltonetworks",
      "global_fw_license": "[parameters('licenseType')]",
      "global_fw_version": "[if(equals(variables('global_fw_license'), 'bundle2-for-ms'), '8.1.00', '8.1.0')]",
      "global_fw_product": "[if(equals(variables('global_fw_license'), 'bundle2-for-ms'), 'vmseries-forms', 'vmseries1')]",
      "global_fw_vmSize": "Standard_DS3_v2",
      "global_fw_enableAcceleratedNetworking": "false",
      "global_fw_avset_option": "Create new availability set",
      "global_fw_avset_name": "vmseries-fw-as",
      "global_fw_avset_id": { "id": "[resourceId('Microsoft.Compute/availabilitySets', variables('global_fw_avset_name'))]"},
      "global_spoke1_resource_group": "[resourceGroup().name]",
      "global_spoke1_vnet_name": "spoke1-vnet",
      "global_spoke1_subnet_name": "spoke1-subnet1",
      "global_spoke2_resource_group": "[resourceGroup().name]",
      "global_spoke2_vnet_name": "spoke2-vnet",
      "global_spoke2_subnet_name": "spoke2-subnet1",
      "global_spoke_vm_vmSize": "Standard_B1s",
      "global_spoke_vm_publisher": "Canonical",
      "global_spoke_vm_offer": "UbuntuServer",
      "global_spoke_vm_sku": "18.04-LTS",
      "global_spoke_vm_version": "latest",
      "global_spoke_vm_osType": "Linux",
      "global_spoke_vm_diskSizeGB": "30",
      "global_spoke_vm_diskType": "Standard_LRS",
      "global_routetable_name": "spoke-route-table",
  
      "COMMENT_nsg": "NSG TEMPLATE VARIABLES",
      "nsg_mgmt_inbound_rule_name": "allow-inbound-https-ssh",
      "nsg_mgmt_inbound_rule_sourceAddress": "0.0.0.0/0",
      "nsg_mgmt_inbound_rule_ports": ["22", "443"],
      "nsg_data_inbound_rule_name": "allow-all-inbound",
      "nsg_data_outbound_rule_name": "allow-all-outbound",
  
      "COMMENT_vnet": "NEW VNET TEMPLATE VARIABLES",
      "vnet_cidr": "10.0.0.0/16",
      "vnet_subnet0_cidr": "10.0.0.0/24",
      "vnet_subnet1_cidr": "10.0.1.0/24",
      "vnet_subnet2_cidr": "10.0.2.0/24",
      "vnet_subnet3_cidr": "10.0.3.0/24",
      "vnet_to_spoke1_peer_name": "transit-to-spoke1",
      "vnet_to_spoke2_peer_name": "transit-to-spoke2",
      "spoke1_vnet_cidr": "172.17.0.0/16",
      "spoke1_subnet_cidr": "172.17.0.0/24",
      "spoke1_to_vnet_peer_name": "spoke1-to-transit",
      "spoke2_vnet_cidr": "192.168.0.0/16",
      "spoke2_subnet_cidr": "192.168.0.0/24",
      "spoke2_to_vnet_peer_name": "spoke2-to-transit",
  
      "COMMENT_publicLb": "PUBLIC LB TEMPLATE VARIABLES",
      "publicLb_name": "vmseries-public-lb",
      "publicLb_resource_group": "[resourceGroup().name]",
      "publicLb_frontend_name": "LoadBalancerFrontEnd",
      "publicLb_pool_name": "LoadBalancerBackendPool",
      "publicLb_rule_name": "rule-",
      "publicLb_rule_delimiters": [",", ";" ],
      "publicLb_rule_ports": "80, 443, 22, 3389",
      "publicLb_rule_port": "[split(variables('publicLb_rule_ports'),variables('publicLb_rule_delimiters'))]",
      "publicLb_rule_protocol": "Tcp",
      "publicLb_rule_enableTcpReset": "false",
      "publicLb_rule_loadDistribution": "Default",
      "publicLb_rule_enableFloatingIP": false,
      "publicLb_rule_disableOutboundSnat": "false",
      "publicLb_probe_name": "HealthProbe",
      "publicLb_probe_port": "22",
      "publicLb_probe_protocol": "Tcp",
      "publicLb_probe_intervalInSeconds": 5,
      "publicLb_probe_numberOfProbes": 2,
      "publicLb_pip_name": "vmseries-public-lb-pip",
  
      "COMMENT_internalLb": "INTERNAL LB TEMPLATE VARIABLES",
      "internalLb_name": "vmseries-internal-lb",
      "internalLb_resource_group": "[resourceGroup().name]",
      "internalLb_frontend_name": "LoadBalancerFrontEnd",
      "internalLb_frontend_ip": "10.0.3.100",
      "internalLb_pool_name": "LoadBalancerBackendPool",
      "internalLb_rule_name": "ha-ports",
      "internalLb_rule_port": 0,
      "internalLb_rule_protocol": "All",
      "internalLb_rule_enableTcpReset": "false",
      "internalLb_rule_loadDistribution": "Default",
      "internalLb_rule_enableFloatingIP": true,
      "internalLb_rule_disableOutboundSnat": "false",
      "internalLb_probe_name": "HealthProbe",
      "internalLb_probe_port": "22",
      "internalLb_probe_protocol": "Tcp",
      "internalLb_probe_intervalInSeconds": 5,
      "internalLb_probe_numberOfProbes": 2,
  
      "COMMENT_avset": "AV SET NESTED TEMPLATE VARIABLES",
      "avset_sku": "Aligned",
      "avset_platformUpdateDomainCount": 5,
      "avset_platformFaultDomainCount": 2,
  
      "COMMENT_spoke1": "SPOKE1 VM VARIABLES",
      "spoke1_vm_name": "spoke1-vm-ubuntu",
      "spoke1_vm_nic_name": "spoke1-vm-nic0",
      "spoke1_vm_nic_ip": "172.17.0.4",
  
      "COMMENT_spoke2": "SPOKE2 VM VARIABLES",
      "spoke2_vm_name": "spoke2-vm-ubuntu",
      "spoke2_vm_nic_name": "spoke2-vm-nic0",
      "spoke2_vm_nic_ip": "192.168.0.4",
  
      "COMMENT_fw1": "FW1 TEMPLATE VARIABLES",
      "fw1_computerName": "vmseries-vm1",
      "fw1_interface0_name": "vmseries-vm1-nic0",
      "fw1_interface1_name": "vmseries-vm1-nic1",
      "fw1_interface2_name": "vmseries-vm1-nic2",
      "fw1_interface0_ip": "10.0.0.4",
      "fw1_interface1_ip": "10.0.1.4",
      "fw1_interface2_ip": "10.0.2.4",
      "fw1_interface0_pip_name": "vmseries-vm1-nic0-pip",
      "fw1_interface1_pip_name": "vmseries-vm1-nic1-pip",
      "fw1_interface0_pip_id": { "id": "[resourceId('Microsoft.Network/publicIPAddresses', variables('fw1_interface0_pip_name'))]" },
      "fw1_interface1_pip_id": { "id": "[resourceId('Microsoft.Network/publicIPAddresses', variables('fw1_interface1_pip_name'))]" },
      
  
      "COMMENT_fw2": "FW2 TEMPLATE VARIABLES",
      "fw2_computerName": "vmseries-vm2",
      "fw2_interface0_name": "vmseries-vm2-nic0",
      "fw2_interface1_name": "vmseries-vm2-nic1",
      "fw2_interface2_name": "vmseries-vm2-nic2",
      "fw2_interface0_ip": "10.0.0.5",
      "fw2_interface1_ip": "10.0.1.5",
      "fw2_interface2_ip": "10.0.2.5",
      "fw2_interface0_pip_name": "vmseries-vm2-nic0-pip",
      "fw2_interface1_pip_name": "vmseries-vm2-nic1-pip",
      "fw2_interface0_pip_id": { "id": "[resourceId('Microsoft.Network/publicIPAddresses', variables('fw2_interface0_pip_name'))]" },
      "fw2_interface1_pip_id": { "id": "[resourceId('Microsoft.Network/publicIPAddresses', variables('fw2_interface1_pip_name'))]" },
  
    },
    "resources": [
      {
        "comments": "CREATE FIREWALL MGMT & DATA NSGS",
        "type": "Microsoft.Resources/deployments",
        "name": "CREATE_NSGS",
        "apiVersion": "[variables('global_var_apiVersion')]",
        "resourceGroup": "[resourceGroup().name]",
        "properties": {
          "mode": "Incremental",
          "template": {
            "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
            "contentVersion": "1.0.0.0",
            "resources": [
              {
                "type": "Microsoft.Network/networkSecurityGroups",
                "name": "[variables('global_fw_mgmtnsg_name')]",
                "apiVersion": "[variables('global_var_apiVersion')]",
                "location": "[resourceGroup().location]",
                "properties": {
                  "securityRules": [
                    {
                      "name": "[variables('nsg_mgmt_inbound_rule_name')]",
                      "properties": {
                        "protocol": "Tcp",
                        "sourcePortRange": "*",
                        "sourceAddressPrefix": "[variables('nsg_mgmt_inbound_rule_sourceAddress')]",
                        "destinationAddressPrefix": "*",
                        "access": "Allow",
                        "priority": "100",
                        "direction": "Inbound",
                        "sourcePortRanges": [],
                        "destinationPortRanges": "[variables('nsg_mgmt_inbound_rule_ports')]",
                        "sourceAddressPrefixes": [],
                        "destinationAddressPrefixes": []
                      }
                    }
                  ]
                }
              },
              {
                "type": "Microsoft.Network/networkSecurityGroups",
                "name": "[variables('global_fw_dataNsg_name')]",
                "apiVersion": "[variables('global_var_apiVersion')]",
                "location": "[resourceGroup().location]",
                "properties": {
                  "securityRules": [
                    {
                      "name": "[variables('nsg_data_inbound_rule_name')]",
                      "properties": {
                        "protocol": "*",
                        "sourcePortRange": "*",
                        "destinationPortRange": "*",
                        "sourceAddressPrefix": "*",
                        "destinationAddressPrefix": "*",
                        "access": "Allow",
                        "priority": "100",
                        "direction": "Inbound",
                        "sourcePortRanges": [],
                        "destinationPortRanges": [],
                        "sourceAddressPrefixes": [],
                        "destinationAddressPrefixes": []
                      }
                    },
                    {
                      "name": "[variables('nsg_data_outbound_rule_name')]",
                      "properties": {
                        "protocol": "*",
                        "sourcePortRange": "*",
                        "destinationPortRange": "*",
                        "sourceAddressPrefix": "*",
                        "destinationAddressPrefix": "*",
                        "access": "Allow",
                        "priority": "100",
                        "direction": "Outbound",
                        "sourcePortRanges": [],
                        "destinationPortRanges": [],
                        "sourceAddressPrefixes": [],
                        "destinationAddressPrefixes": []
                      }
                    }
                  ]
                }
              }
            ]
          }
        }
      },
      {
        "comments": "CREATE ROUTE TABLE FOR SPOKE SUBNETS",
        "type": "Microsoft.Resources/deployments",
        "name": "CREATE_ROUTE_TABLE",
        "apiVersion": "[variables('global_var_apiVersion')]",
        "resourceGroup": "[variables('global_var_resource_group')]",
        "properties": {
          "mode": "Incremental",
          "template": {
            "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
            "contentVersion": "1.0.0.0",
            "resources": [
              {
                "type": "Microsoft.Network/routeTables",
                "name": "[variables('global_routetable_name')]",
                "apiVersion": "[variables('global_var_apiVersion')]",
                "location": "[resourceGroup().location]",
                "properties": {
                  "routes": [
                    {
                      "name": "default-udr",
                      "properties": {
                        "addressPrefix": "0.0.0.0/0",
                        "nextHopType": "VirtualAppliance",
                        "nextHopIpAddress": "[variables('internalLb_frontend_ip')]"
                      }
                    },
                    {
                      "name": "spoke1-udr",
                      "properties": {
                        "addressPrefix": "[variables('spoke1_vnet_cidr')]",
                        "nextHopType": "VirtualAppliance",
                        "nextHopIpAddress": "[variables('internalLb_frontend_ip')]"
                      }
                    },
                    {
                      "name": "spoke2-udr",
                      "properties": {
                        "addressPrefix": "[variables('spoke2_vnet_cidr')]",
                        "nextHopType": "VirtualAppliance",
                        "nextHopIpAddress": "[variables('internalLb_frontend_ip')]"
                      }
                    }
                  ]
                }
              }
            ]
          }
        }
      },
      {
        "comments": "CREATE VIRTUAL NETWORK WITH 4 SUBNETS",
        "type": "Microsoft.Resources/deployments",
        "name": "CREATE_VNET",
        "apiVersion": "[variables('global_var_apiVersion')]",
        "resourceGroup": "[variables('global_vnet_resource_group')]",
        "dependsOn": [
          "CREATE_ROUTE_TABLE"
        ],
        "condition": "[equals(variables('global_vnet_option'), 'Create new VNET')]",
        "properties": {
          "mode": "Incremental",
          "template": {
            "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
            "contentVersion": "1.0.0.0",
            "resources": [
              {
                "type": "Microsoft.Network/virtualNetworks",
                "name": "[variables('global_vnet_name')]",
                "apiVersion": "[variables('global_var_apiVersion')]",
                "location": "[resourceGroup().location]",
                "properties": {
                  "addressSpace": {
                    "addressPrefixes": [
                      "[variables('vnet_cidr')]"
                    ]
                  },
                  "subnets": [
                    {
                      "name": "[variables('global_vnet_subnet0_name')]",
                      "properties": {
                        "addressPrefix": "[variables('vnet_subnet0_cidr')]"
                      }
                    },
                    {
                      "name": "[variables('global_vnet_subnet1_name')]",
                      "properties": {
                        "addressPrefix": "[variables('vnet_subnet1_cidr')]"
                      }
                    },
                    {
                      "name": "[variables('global_vnet_subnet2_name')]",
                      "properties": {
                        "addressPrefix": "[variables('vnet_subnet2_cidr')]"
                      }
                    },
                    {
                      "name": "[variables('global_vnet_subnet3_name')]",
                      "properties": {
                        "addressPrefix": "[variables('vnet_subnet3_cidr')]"
                      }
                    }
                  ]
                },
                "resources": [
                  {
                    "type": "virtualNetworkPeerings",
                    "name": "[variables('vnet_to_spoke1_peer_name')]",
                    "apiVersion": "[variables('global_var_apiVersion')]",
                    "location": "[variables('global_vnet_resource_group')",
                    "dependsOn": [
                      "[concat('Microsoft.Network/virtualNetworks/', variables('global_vnet_name'))]",
                      "[concat('Microsoft.Network/virtualNetworks/', variables('global_spoke1_vnet_name'))]"
                    ],
                    "properties": {
                      "allowVirtualNetworkAccess": "true",
                      "allowForwardedTraffic": "true",
                      "allowGatewayTransit": "false",
                      "useRemoteGateways": "false",
                      "remoteVirtualNetwork": {
                        "id": "[resourceId('Microsoft.Network/virtualNetworks',variables('global_spoke1_vnet_name'))]"
                      }
                    }
                  },
                  {
                    "type": "virtualNetworkPeerings",
                    "name": "[variables('vnet_to_spoke2_peer_name')]",
                    "apiVersion": "[variables('global_var_apiVersion')]",
                    "location": "[variables('global_vnet_resource_group')",
                    "dependsOn": [
                      "[concat('Microsoft.Network/virtualNetworks/', variables('global_vnet_name'))]",
                      "[concat('Microsoft.Network/virtualNetworks/', variables('global_spoke2_vnet_name'))]"
                    ],
                    "properties": {
                      "allowVirtualNetworkAccess": "true",
                      "allowForwardedTraffic": "true",
                      "allowGatewayTransit": "false",
                      "useRemoteGateways": "false",
                      "remoteVirtualNetwork": {
                        "id": "[resourceId('Microsoft.Network/virtualNetworks',variables('global_spoke2_vnet_name'))]"
                      }
                    }
                  }
                ]
              },
              {
                "type": "Microsoft.Network/virtualNetworks",
                "name": "[variables('global_spoke1_vnet_name')]",
                "apiVersion": "[variables('global_var_apiVersion')]",
                "location": "[resourceGroup().location]",
                "properties": {
                  "addressSpace": {
                    "addressPrefixes": [
                      "[variables('spoke1_vnet_cidr')]"
                    ]
                  },
                  "subnets": [
                    {
                      "name": "[variables('global_spoke1_subnet_name')]",
                      "properties": {
                        "addressPrefix": "[variables('spoke1_subnet_cidr')]",
                        "routeTable": {
                          "id": "[resourceId('Microsoft.Network/routeTables', variables('global_routetable_name'))]"
                        }
                      }
                    }
                  ]
                },
                "resources": [
                  {
                    "type": "virtualNetworkPeerings",
                    "name": "[variables('spoke1_to_vnet_peer_name')]",
                    "apiVersion": "[variables('global_var_apiVersion')]",
                    "location": "[variables('global_spoke1_resource_group')",
                    "dependsOn": [
                      "[concat('Microsoft.Network/virtualNetworks/', variables('global_vnet_name'))]",
                      "[concat('Microsoft.Network/virtualNetworks/', variables('global_spoke1_vnet_name'))]"
                    ],
                    "properties": {
                      "allowVirtualNetworkAccess": "true",
                      "allowForwardedTraffic": "true",
                      "allowGatewayTransit": "false",
                      "useRemoteGateways": "false",
                      "remoteVirtualNetwork": {
                        "id": "[resourceId('Microsoft.Network/virtualNetworks',variables('global_vnet_name'))]"
                      }
                    }
                  }
                ]
              },
              {
                "type": "Microsoft.Network/virtualNetworks",
                "name": "[variables('global_spoke2_vnet_name')]",
                "apiVersion": "[variables('global_var_apiVersion')]",
                "location": "[resourceGroup().location]",
                "properties": {
                  "addressSpace": {
                    "addressPrefixes": [
                      "[variables('spoke2_vnet_cidr')]"
                    ]
                  },
                  "subnets": [
                    {
                      "name": "[variables('global_spoke2_subnet_name')]",
                      "properties": {
                        "addressPrefix": "[variables('spoke2_subnet_cidr')]",
                        "routeTable": {
                          "id": "[resourceId('Microsoft.Network/routeTables', variables('global_routetable_name'))]"
                        }
                      }
                    }
                  ]
                },
                "resources": [
                  {
                    "type": "virtualNetworkPeerings",
                    "name": "[variables('spoke2_to_vnet_peer_name')]",
                    "apiVersion": "[variables('global_var_apiVersion')]",
                    "location": "[variables('global_spoke1_resource_group')",
                    "dependsOn": [
                      "[concat('Microsoft.Network/virtualNetworks/', variables('global_vnet_name'))]",
                      "[concat('Microsoft.Network/virtualNetworks/', variables('global_spoke2_vnet_name'))]"
                    ],
                    "properties": {
                      "allowVirtualNetworkAccess": "true",
                      "allowForwardedTraffic": "true",
                      "allowGatewayTransit": "false",
                      "useRemoteGateways": "false",
                      "remoteVirtualNetwork": {
                        "id": "[resourceId('Microsoft.Network/virtualNetworks',variables('global_vnet_name'))]"
                      }
                    }
                  }
                ]
              }
            ]
          }
        }
      },
      {
        "comments": "CREATE FIREWALL PUBLIC LOAD BALANCER",
        "type": "Microsoft.Resources/deployments",
        "name": "CREATE_PUBLICLB",
        "apiVersion": "[variables('global_var_apiVersion')]",
        "resourceGroup": "[variables('publicLb_resource_group')]",
        "properties": {
          "mode": "Incremental",
          "template": {
            "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
            "contentVersion": "1.0.0.0",
            "resources": [
              {
                "type": "Microsoft.Network/publicIPAddresses",
                "sku": {
                  "name": "[variables('global_var_sku')]",
                  "tier": "[variables('global_var_tier')]"
                },
                "name": "[variables('publicLb_pip_name')]",
                "apiVersion": "[variables('global_var_apiVersion')]",
                "location": "[resourceGroup().location]",
                "properties": {
                  "publicIPAddressVersion": "[variables('global_var_networkVersion')]",
                  "publicIPAllocationMethod": "[variables('global_var_allocationMethod')]",
                  "idleTimeoutInMinutes": "[variables('global_var_idleTimeoutInMinutes')]"
                }
              },
              {
                "type": "Microsoft.Network/loadBalancers",
                "sku": {
                  "name": "[variables('global_var_sku')]"
                },
                "name": "[variables('publicLb_name')]",
                "apiVersion": "[variables('global_var_apiVersion')]",
                "location": "[resourceGroup().location]",
                "properties": {
                  "frontendIPConfigurations": [
                    {
                      "name": "[variables('publicLb_frontend_name')]",
                      "properties": {
                        "publicIPAddress": {
                          "id": "[resourceId(variables('publicLb_resource_group'), 'Microsoft.Network/publicIPAddresses', variables('publicLb_pip_name'))]"
                        }
                      }
                    }
                  ],
                  "backendAddressPools": [
                    {
                      "name": "[variables('publicLb_pool_name')]"
                    }
                  ],
                  "copy": [
                    {
                      "name": "loadBalancingRules",
                      "count": "[length(variables('publicLb_rule_port'))]",
                      "input": {
                        "name": "[concat(variables('publicLb_rule_name'), copyIndex('loadBalancingRules'))]",
                        "properties": {
                          "frontendIPConfiguration": {
                            "id": "[concat(resourceId(variables('publicLb_resource_group'), 'Microsoft.Network/loadBalancers', variables('publicLb_name')), '/frontendIPConfigurations/', variables('publicLb_frontend_name'))]"
                          },
                          "frontendPort": "[variables('publicLb_rule_port')[copyIndex('loadBalancingRules')]]",
                          "backendPort": "[variables('publicLb_rule_port')[copyIndex('loadBalancingRules')]]",
                          "enableFloatingIP": "[variables('publicLb_rule_enableFloatingIP')]",
                          "idleTimeoutInMinutes": "[variables('global_var_idleTimeoutInMinutes')]",
                          "protocol": "[variables('publicLb_rule_protocol')]",
                          "enableTcpReset": "[variables('publicLb_rule_enableTcpReset')]",
                          "loadDistribution": "[variables('publicLb_rule_loadDistribution')]",
                          "disableOutboundSnat": "[variables('publicLb_rule_disableOutboundSnat')]",
                          "backendAddressPool": {
                            "id": "[concat(resourceId(variables('publicLb_resource_group'), 'Microsoft.Network/loadBalancers', variables('publicLb_name')), '/backendAddressPools/', variables('publicLb_pool_name'))]"
                          },
                          "probe": {
                            "id": "[concat(resourceId(variables('publicLb_resource_group'), 'Microsoft.Network/loadBalancers', variables('publicLb_name')), '/probes/', variables('publicLb_probe_name'))]"
                          }
                        }
                      }
                    }
                  ],
                  "probes": [
                    {
                      "name": "[variables('publicLb_probe_name')]",
                      "properties": {
                        "protocol": "[variables('publicLb_probe_protocol')]",
                        "port": "[variables('publicLb_probe_port')]",
                        "intervalInSeconds": "[variables('publicLb_probe_intervalInSeconds')]",
                        "numberOfProbes": "[variables('publicLb_probe_numberOfProbes')]"
                      }
                    }
                  ]
                },
                "dependsOn": [
                  "[resourceId(variables('publicLb_resource_group'), 'Microsoft.Network/publicIPAddresses', variables('publicLb_pip_name'))]"
                ]
              }
            ]
          }
        }
      },
      {
        "comments": "CREATE FIREWALL INTERNAL LOAD BALANCER WITH HA PORTS",
        "type": "Microsoft.Resources/deployments",
        "name": "CREATE_INTERNALLB",
        "apiVersion": "[variables('global_var_apiVersion')]",
        "resourceGroup": "[variables('internalLb_resource_group')]",
        "dependsOn": [
          "CREATE_VNET"
        ],
        "properties": {
          "mode": "Incremental",
          "template": {
            "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
            "contentVersion": "1.0.0.0",
            "resources": [
              {
                "type": "Microsoft.Network/loadBalancers",
                "sku": {
                  "name": "[variables('global_var_sku')]",
                  "tier": "[variables('global_var_tier')]"
                },
                "name": "[variables('internalLb_name')]",
                "apiVersion": "[variables('global_var_apiVersion')]",
                "location": "[resourceGroup().location]",
                "properties": {
                  "frontendIPConfigurations": [
                    {
                      "name": "[variables('internalLb_frontend_name')]",
                      "properties": {
                        "privateIPAddress": "[variables('internalLb_frontend_ip')]",
                        "privateIPAllocationMethod": "[variables('global_var_allocationMethod')]",
                        "subnet": {
                          "id": "[concat(resourceId(variables('global_vnet_resource_group'), 'Microsoft.Network/virtualNetworks', variables('global_vnet_name')), '/subnets/', variables('global_vnet_subnet3_name'))]"
                        }
                      }
                    }
                  ],
                  "backendAddressPools": [
                    {
                      "name": "[variables('internalLb_pool_name')]"
                    }
                  ],
                  "loadBalancingRules": [
                    {
                      "name": "[variables('internalLb_rule_name')]",
                      "properties": {
                        "frontendIPConfiguration": {
                          "id": "[concat(resourceId(variables('internalLb_resource_group'), 'Microsoft.Network/loadBalancers', variables('internalLb_name')), '/frontendIPConfigurations/', variables('internalLb_frontend_name'))]"
                        },
                        "frontendPort": "[variables('internalLb_rule_port')]",
                        "backendPort": "[variables('internalLb_rule_port')]",
                        "enableFloatingIP": "[variables('internalLb_rule_enableFloatingIP')]",
                        "idleTimeoutInMinutes": "[variables('global_var_idleTimeoutInMinutes')]",
                        "protocol": "[variables('internalLb_rule_protocol')]",
                        "enableTcpReset": "[variables('internalLb_rule_enableTcpReset')]",
                        "loadDistribution": "[variables('internalLb_rule_loadDistribution')]",
                        "disableOutboundSnat": "[variables('internalLb_rule_disableOutboundSnat')]",
                        "backendAddressPool": {
                          "id": "[concat(resourceId(variables('internalLb_resource_group'), 'Microsoft.Network/loadBalancers', variables('internalLb_name')), '/backendAddressPools/', variables('internalLb_pool_name'))]"
                        },
                        "probe": {
                          "id": "[concat(resourceId(variables('internalLb_resource_group'), 'Microsoft.Network/loadBalancers', variables('internalLb_name')), '/probes/', variables('internalLb_probe_name'))]"
                        }
                      }
                    }
                  ],
                  "probes": [
                    {
                      "name": "[variables('internalLb_probe_name')]",
                      "properties": {
                        "protocol": "[variables('internalLb_probe_protocol')]",
                        "port": "[variables('internalLb_probe_port')]",
                        "intervalInSeconds": "[variables('internalLb_probe_intervalInSeconds')]",
                        "numberOfProbes": "[variables('internalLb_probe_numberOfProbes')]"
                      }
                    }
                  ]
                }
              }
            ]
          }
        }
      },
      {
        "comments": "CREATE FIREWALL AVAILABILITY SET",
        "type": "Microsoft.Resources/deployments",
        "name": "CREATE_AVSET",
        "apiVersion": "[variables('global_var_apiVersion')]",
        "resourceGroup": "[resourceGroup().name]",
        "condition": "[equals(variables('global_fw_avset_option'), 'Create new availability set')]",
        "properties": {
          "mode": "Incremental",
          "template": {
            "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
            "contentVersion": "1.0.0.0",
            "resources": [
              {
                "type": "Microsoft.Compute/availabilitySets",
                "sku": {
                  "name": "[variables('avset_sku')]"
                },
                "name": "[variables('global_fw_avset_name')]",
                "apiVersion": "[variables('global_var_apiVersion')]",
                "location": "[resourceGroup().location]",
                "properties": {
                  "platformUpdateDomainCount": "[variables('avset_platformUpdateDomainCount')]",
                  "platformFaultDomainCount": "[variables('avset_platformFaultDomainCount')]"
                }
              }
            ]
          }
        }
      },
      {
        "comments": "CREATE SPOKE1 VM WITH NIC",
        "type": "Microsoft.Resources/deployments",
        "name": "CREATE_SPOKE1_VM",
        "apiVersion": "[variables('global_var_apiVersion')]",
        "resourceGroup": "[variables('global_spoke1_resource_group')]",
        "dependsOn": [
          "CREATE_VNET"
        ],
        "properties": {
          "mode": "Incremental",
          "template": {
            "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
            "contentVersion": "1.0.0.0",
            "resources": [
              {
                "comments": "CREATE_SPOKE1_VM_NIC",
                "type": "Microsoft.Network/networkInterfaces",
                "name": "[variables('spoke1_vm_nic_name')]",
                "apiVersion": "[variables('global_var_apiVersion')]",
                "location": "[resourceGroup().location]",
                "properties": {
                  "ipConfigurations": [
                    {
                      "name": "ipconfig1",
                      "properties": {
                        "privateIpAddress": "[variables('spoke1_vm_nic_ip')]",
                        "privateIPAllocationMethod": "[variables('global_var_allocationMethod')]",
                        "subnet": {
                          "id": "[concat(resourceId('Microsoft.Network/virtualNetworks', variables('global_spoke1_vnet_name')), '/subnets/', variables('global_spoke1_subnet_name'))]"
                        },
                        "primary": true,
                        "privateIPAddressVersion": "[variables('global_var_networkVersion')]"
                      }
                    }
                  ],
                  "enableAcceleratedNetworking": "[variables('global_fw_enableAcceleratedNetworking')]",
                  "enableIPForwarding": false
                }
              },
              {
                "comments": "CREATE_SPOKE1_VM",
                "type": "Microsoft.Compute/virtualMachines",
                "name": "[variables('spoke1_vm_name')]",
                "apiVersion": "[variables('global_var_apiVersion')]",
                "location": "[resourceGroup().location]",
                "properties": {
                  "hardwareProfile": {
                    "vmSize": "[variables('global_spoke_vm_vmSize')]"
                  },
                  "storageProfile": {
                    "imageReference": {
                      "publisher": "[variables('global_spoke_vm_publisher')]",
                      "offer": "[variables('global_spoke_vm_offer')]",
                      "sku": "[variables('global_spoke_vm_sku')]",
                      "version": "[variables('global_spoke_vm_version')]"
                    },
                    "osDisk": {
                      "osType": "[variables('global_spoke_vm_osType')]",
                      "createOption": "FromImage",
                      "caching": "ReadWrite",
                      "managedDisk": {
                        "storageAccountType": "[variables('global_spoke_vm_diskType')]"
                      },
                      "diskSizeGB": "[variables('global_spoke_vm_diskSizeGB')]"
                    }
                  },
                  "osProfile": {
                    "computerName": "[variables('spoke1_vm_name')]",
                    "adminUsername": "[variables('global_fw_adminUsername')]",
                    "adminPassword": "[variables('global_fw_adminPassword')]",
                    "linuxConfiguration": {
                      "disablePasswordAuthentication": false,
                      "provisionVMAgent": true
                    },
                    "allowExtensionOperations": true
                  },
                  "networkProfile": {
                    "networkInterfaces": [
                      {
                        "id": "[resourceId('Microsoft.Network/networkInterfaces', variables('spoke1_vm_nic_name'))]",
                        "properties": {
                          "primary": true
                        }
                      }
                    ]
                  }
                },
                "dependsOn": [
                  "[resourceId('Microsoft.Network/networkInterfaces', variables('spoke1_vm_nic_name'))]"
                ]
              }
            ]
          }
        }
      },
      {
        "comments": "CREATE SPOKE2 VM WITH NIC",
        "type": "Microsoft.Resources/deployments",
        "name": "CREATE_SPOKE2_VM",
        "apiVersion": "[variables('global_var_apiVersion')]",
        "resourceGroup": "[variables('global_spoke2_resource_group')]",
        "dependsOn": [
          "CREATE_VNET"
        ],
        "properties": {
          "mode": "Incremental",
          "template": {
            "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
            "contentVersion": "1.0.0.0",
            "resources": [
              {
                "comments": "CREATE_SPOKE2_VM_NIC",
                "type": "Microsoft.Network/networkInterfaces",
                "name": "[variables('spoke2_vm_nic_name')]",
                "apiVersion": "[variables('global_var_apiVersion')]",
                "location": "[resourceGroup().location]",
                "properties": {
                  "ipConfigurations": [
                    {
                      "name": "ipconfig1",
                      "properties": {
                        "privateIpAddress": "[variables('spoke2_vm_nic_ip')]",
                        "privateIPAllocationMethod": "[variables('global_var_allocationMethod')]",
                        "subnet": {
                          "id": "[concat(resourceId('Microsoft.Network/virtualNetworks', variables('global_spoke2_vnet_name')), '/subnets/', variables('global_spoke2_subnet_name'))]"
                        },
                        "primary": true,
                        "privateIPAddressVersion": "[variables('global_var_networkVersion')]"
                      }
                    }
                  ],
                  "enableAcceleratedNetworking": "[variables('global_fw_enableAcceleratedNetworking')]",
                  "enableIPForwarding": false
                }
              },
              {
                "comments": "CREATE_SPOKE2_VM",
                "type": "Microsoft.Compute/virtualMachines",
                "name": "[variables('spoke2_vm_name')]",
                "apiVersion": "[variables('global_var_apiVersion')]",
                "location": "[resourceGroup().location]",
                "properties": {
                  "hardwareProfile": {
                    "vmSize": "[variables('global_spoke_vm_vmSize')]"
                  },
                  "storageProfile": {
                    "imageReference": {
                      "publisher": "[variables('global_spoke_vm_publisher')]",
                      "offer": "[variables('global_spoke_vm_offer')]",
                      "sku": "[variables('global_spoke_vm_sku')]",
                      "version": "[variables('global_spoke_vm_version')]"
                    },
                    "osDisk": {
                      "osType": "[variables('global_spoke_vm_osType')]",
                      "createOption": "FromImage",
                      "caching": "ReadWrite",
                      "managedDisk": {
                        "storageAccountType": "[variables('global_spoke_vm_diskType')]"
                      },
                      "diskSizeGB": "[variables('global_spoke_vm_diskSizeGB')]"
                    }
                  },
                  "osProfile": {
                    "computerName": "[variables('spoke2_vm_name')]",
                    "adminUsername": "[variables('global_fw_adminUsername')]",
                    "adminPassword": "[variables('global_fw_adminPassword')]",
                    "linuxConfiguration": {
                      "disablePasswordAuthentication": false,
                      "provisionVMAgent": true
                    },
                    "allowExtensionOperations": true
                  },
                  "networkProfile": {
                    "networkInterfaces": [
                      {
                        "id": "[resourceId('Microsoft.Network/networkInterfaces', variables('spoke2_vm_nic_name'))]",
                        "properties": {
                          "primary": true
                        }
                      }
                    ]
                  }
                },
                "dependsOn": [
                  "[resourceId('Microsoft.Network/networkInterfaces', variables('spoke2_vm_nic_name'))]"
                ]
              }
            ]
          }
        }
      },
      {
        "comments": "CREATE FW1 WITH INTERFACES",
        "type": "Microsoft.Resources/deployments",
        "name": "CREATE_FW1",
        "apiVersion": "[variables('global_var_apiVersion')]",
        "resourceGroup": "[resourceGroup().name]",
        "dependsOn": [
          "CREATE_VNET",
          "CREATE_AVSET",
          "CREATE_NSGS",
          "CREATE_INTERNALLB",
          "CREATE_PUBLICLB"
        ],
        "properties": {
          "mode": "Incremental",
          "template": {
            "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
            "contentVersion": "1.0.0.0",
            "resources": [
              {
                "condition": "[equals(variables('global_fw_interface0_pip_option'), 'Yes')]",
                "type": "Microsoft.Network/publicIPAddresses",
                "sku": {
                  "name": "[variables('global_var_sku')]",
                  "tier": "[variables('global_var_tier')]"
                },
                "name": "[variables('fw1_interface0_pip_name')]",
                "apiVersion": "[variables('global_var_apiVersion')]",
                "location": "[resourceGroup().location]",
                "properties": {
                  "publicIPAddressVersion": "[variables('global_var_networkVersion')]",
                  "publicIPAllocationMethod": "[variables('global_var_allocationMethod')]",
                  "idleTimeoutInMinutes": "[variables('global_var_idleTimeoutInMinutes')]"
                }
              },
              {
                "condition": "[equals(variables('global_fw_interface1_pip_option'), 'Yes')]",
                "type": "Microsoft.Network/publicIPAddresses",
                "sku": {
                  "name": "[variables('global_var_sku')]",
                  "tier": "[variables('global_var_tier')]"
                },
                "name": "[variables('fw1_interface1_pip_name')]",
                "apiVersion": "[variables('global_var_apiVersion')]",
                "location": "[resourceGroup().location]",
                "properties": {
                  "publicIPAddressVersion": "[variables('global_var_networkVersion')]",
                  "publicIPAllocationMethod": "[variables('global_var_allocationMethod')]",
                  "idleTimeoutInMinutes": "[variables('global_var_idleTimeoutInMinutes')]"
                }
              },
              {
                "type": "Microsoft.Network/networkInterfaces",
                "name": "[variables('fw1_interface0_name')]",
                "apiVersion": "[variables('global_var_apiVersion')]",
                "location": "[resourceGroup().location]",
                "properties": {
                  "ipConfigurations": [
                    {
                      "name": "ipconfig1",
                      "properties": {
                        "privateIpAddress": "[variables('fw1_interface0_ip')]",
                        "privateIPAllocationMethod": "[variables('global_var_allocationMethod')]",
                        "publicIpAddress": "[if(equals(variables('global_fw_interface0_pip_option'), 'Yes'), variables('fw1_interface0_pip_id'), json('null'))]",
                        "subnet": {
                          "id": "[concat(resourceId(variables('global_vnet_resource_group'), 'Microsoft.Network/virtualNetworks', variables('global_vnet_name')), '/subnets/', variables('global_vnet_subnet0_name'))]"
                        },
                        "primary": true,
                        "privateIPAddressVersion": "[variables('global_var_networkVersion')]"
                      }
                    }
                  ],
                  "enableAcceleratedNetworking": "[variables('global_fw_enableAcceleratedNetworking')]",
                  "enableIPForwarding": false,
                  "networkSecurityGroup": {
                    "id": "[resourceId('Microsoft.Network/networkSecurityGroups', variables('global_fw_mgmtnsg_name'))]"
                  },
                  "tapConfigurations": []
                },
                "dependsOn": [
                  "[resourceId('Microsoft.Network/publicIPAddresses/', variables('fw1_interface0_pip_name'))]"
                ]
              },
              {
                "type": "Microsoft.Network/networkInterfaces",
                "name": "[variables('fw1_interface1_name')]",
                "apiVersion": "[variables('global_var_apiVersion')]",
                "location": "[resourceGroup().location]",
                "properties": {
                  "ipConfigurations": [
                    {
                      "name": "ipconfig1",
                      "properties": {
                        "privateIpAddress": "[variables('fw1_interface1_ip')]",
                        "privateIPAllocationMethod": "[variables('global_var_allocationMethod')]",
                        "publicIpAddress": "[if(equals(variables('global_fw_interface1_pip_option'), 'Yes'), variables('fw1_interface1_pip_id'), json('null'))]",
                        "subnet": {
                          "id": "[concat(resourceId(variables('global_vnet_resource_group'), 'Microsoft.Network/virtualNetworks', variables('global_vnet_name')), '/subnets/', variables('global_vnet_subnet1_name'))]"
                        },
                        "primary": true,
                        "privateIPAddressVersion": "[variables('global_var_networkVersion')]",
                        "loadBalancerBackendAddressPools": [
                          {
                            "id": "[concat(resourceId(variables('publicLb_resource_group'), 'Microsoft.Network/loadBalancers', variables('publicLb_name')), '/backendAddressPools/', variables('publicLb_pool_name'))]"
                          }
                        ]
                      }
                    }
                  ],
                  "enableAcceleratedNetworking": "[variables('global_fw_enableAcceleratedNetworking')]",
                  "enableIPForwarding": true,
                  "networkSecurityGroup": {
                    "id": "[resourceId('Microsoft.Network/networkSecurityGroups', variables('global_fw_dataNsg_name'))]"
                  },
                  "tapConfigurations": []
                },
                "dependsOn": [
                  "[resourceId('Microsoft.Network/publicIPAddresses/', variables('fw1_interface1_pip_name'))]"
                ]
              },
              {
                "type": "Microsoft.Network/networkInterfaces",
                "name": "[variables('fw1_interface2_name')]",
                "apiVersion": "[variables('global_var_apiVersion')]",
                "location": "[resourceGroup().location]",
                "properties": {
                  "ipConfigurations": [
                    {
                      "name": "ipconfig1",
                      "properties": {
                        "privateIpAddress": "[variables('fw1_interface2_ip')]",
                        "privateIPAllocationMethod": "[variables('global_var_allocationMethod')]",
                        "subnet": {
                          "id": "[concat(resourceId(variables('global_vnet_resource_group'), 'Microsoft.Network/virtualNetworks', variables('global_vnet_name')), '/subnets/', variables('global_vnet_subnet2_name'))]"
                        },
                        "primary": true,
                        "privateIPAddressVersion": "[variables('global_var_networkVersion')]",
                        "loadBalancerBackendAddressPools": [
                          {
                            "id": "[concat(resourceId(variables('internalLb_resource_group'), 'Microsoft.Network/loadBalancers', variables('internalLb_name')), '/backendAddressPools/', variables('internalLb_pool_name'))]"
                          }
                        ]
                      }
                    }
                  ],
                  "enableAcceleratedNetworking": "[variables('global_fw_enableAcceleratedNetworking')]",
                  "enableIPForwarding": true,
                  "networkSecurityGroup": {
                    "id": "[resourceId('Microsoft.Network/networkSecurityGroups', variables('global_fw_dataNsg_name'))]"
                  },
                  "tapConfigurations": []
                }
              },
              {
                "type": "Microsoft.Compute/virtualMachines",
                "name": "[variables('fw1_computerName')]",
                "apiVersion": "[variables('global_var_apiVersion')]",
                "location": "[resourceGroup().location]",
                "plan": {
                  "name": "[variables('global_fw_license')]",
                  "product": "[variables('global_fw_product')]",
                  "publisher": "[variables('global_fw_publisher')]"
                },
                "properties": {
                  "availabilitySet": "[if(equals(variables('global_fw_avset_option'), 'Do not use availability set'), json('null'), variables('global_fw_avset_id'))]",
                  "hardwareProfile": {
                    "vmSize": "[variables('global_fw_vmSize')]"
                  },
                  "storageProfile": {
                    "imageReference": {
                      "publisher": "[variables('global_fw_publisher')]",
                      "offer": "[variables('global_fw_product')]",
                      "sku": "[variables('global_fw_license')]",
                      "version": "[variables('global_fw_version')]"
                    },
                    "osDisk": {
                      "createOption": "FromImage",
                      "caching": "ReadWrite",
                      "managedDisk": {
                        "storageAccountType": "[variables('global_fw_storageAccountType')]"
                      },
                      "diskSizeGB": "[variables('global_fw_diskSizeGB')]"
                    }
                  },
                  "osProfile": {
                    "computerName": "[variables('fw1_computerName')]",
                    "adminUsername": "[variables('global_fw_adminUsername')]",
                    "adminPassword": "[variables('global_fw_adminPassword')]",
                    "customData": "[base64(variables('global_fw_bootstrap'))]"
                  },
                  "networkProfile": {
                    "networkInterfaces": [
                      {
                        "id": "[resourceId('Microsoft.Network/networkInterfaces', variables('fw1_interface0_name'))]",
                        "properties": {
                          "primary": true
                        }
                      },
                      {
                        "id": "[resourceId('Microsoft.Network/networkInterfaces', variables('fw1_interface1_name'))]",
                        "properties": {
                          "primary": false
                        }
                      },
                      {
                        "id": "[resourceId('Microsoft.Network/networkInterfaces', variables('fw1_interface2_name'))]",
                        "properties": {
                          "primary": false
                        }
                      }
                    ]
                  }
                },
                "dependsOn": [
                  "[resourceId('Microsoft.Network/networkInterfaces', variables('fw1_interface0_name'))]",
                  "[resourceId('Microsoft.Network/networkInterfaces', variables('fw1_interface1_name'))]",
                  "[resourceId('Microsoft.Network/networkInterfaces', variables('fw1_interface2_name'))]"
                ]
              }
            ]
          }
        }
      },
      {
        "comments": "CREATE FW2 WITH INTERFACES",
        "type": "Microsoft.Resources/deployments",
        "name": "CREATE_FW2",
        "apiVersion": "[variables('global_var_apiVersion')]",
        "resourceGroup": "[resourceGroup().name]",
        "dependsOn": [
          "CREATE_VNET",
          "CREATE_AVSET",
          "CREATE_NSGS",
          "CREATE_INTERNALLB",
          "CREATE_PUBLICLB"
        ],
        "properties": {
          "mode": "Incremental",
          "template": {
            "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
            "contentVersion": "1.0.0.0",
            "resources": [
              {
                "condition": "[equals(variables('global_fw_interface0_pip_option'), 'Yes')]",
                "type": "Microsoft.Network/publicIPAddresses",
                "sku": {
                  "name": "[variables('global_var_sku')]",
                  "tier": "[variables('global_var_tier')]"
                },
                "name": "[variables('fw2_interface0_pip_name')]",
                "apiVersion": "[variables('global_var_apiVersion')]",
                "location": "[resourceGroup().location]",
                "properties": {
                  "publicIPAddressVersion": "[variables('global_var_networkVersion')]",
                  "publicIPAllocationMethod": "[variables('global_var_allocationMethod')]",
                  "idleTimeoutInMinutes": "[variables('global_var_idleTimeoutInMinutes')]"
                }
              },
              {
                "condition": "[equals(variables('global_fw_interface1_pip_option'), 'Yes')]",
                "type": "Microsoft.Network/publicIPAddresses",
                "sku": {
                  "name": "[variables('global_var_sku')]",
                  "tier": "[variables('global_var_tier')]"
                },
                "name": "[variables('fw2_interface1_pip_name')]",
                "apiVersion": "[variables('global_var_apiVersion')]",
                "location": "[resourceGroup().location]",
                "properties": {
                  "publicIPAddressVersion": "[variables('global_var_networkVersion')]",
                  "publicIPAllocationMethod": "[variables('global_var_allocationMethod')]",
                  "idleTimeoutInMinutes": "[variables('global_var_idleTimeoutInMinutes')]"
                }
              },
              {
                "type": "Microsoft.Network/networkInterfaces",
                "name": "[variables('fw2_interface0_name')]",
                "apiVersion": "[variables('global_var_apiVersion')]",
                "location": "[resourceGroup().location]",
                "properties": {
                  "ipConfigurations": [
                    {
                      "name": "ipconfig1",
                      "properties": {
                        "privateIpAddress": "[variables('fw2_interface0_ip')]",
                        "privateIPAllocationMethod": "[variables('global_var_allocationMethod')]",
                        "publicIpAddress": "[if(equals(variables('global_fw_interface0_pip_option'),'Yes'), variables('fw2_interface0_pip_id'), json('null'))]",
                        "subnet": {
                          "id": "[concat(resourceId(variables('global_vnet_resource_group'), 'Microsoft.Network/virtualNetworks', variables('global_vnet_name')), '/subnets/', variables('global_vnet_subnet0_name'))]"
                        },
                        "primary": true,
                        "privateIPAddressVersion": "[variables('global_var_networkVersion')]"
                      }
                    }
                  ],
                  "enableAcceleratedNetworking": "[variables('global_fw_enableAcceleratedNetworking')]",
                  "enableIPForwarding": false,
                  "networkSecurityGroup": {
                    "id": "[resourceId('Microsoft.Network/networkSecurityGroups', variables('global_fw_mgmtnsg_name'))]"
                  },
                  "tapConfigurations": []
                },
                "dependsOn": [
                  "[resourceId('Microsoft.Network/publicIPAddresses/', variables('fw2_interface0_pip_name'))]"
                ]
              },
              {
                "type": "Microsoft.Network/networkInterfaces",
                "name": "[variables('fw2_interface1_name')]",
                "apiVersion": "[variables('global_var_apiVersion')]",
                "location": "[resourceGroup().location]",
                "properties": {
                  "ipConfigurations": [
                    {
                      "name": "ipconfig1",
                      "properties": {
                        "privateIpAddress": "[variables('fw2_interface1_ip')]",
                        "privateIPAllocationMethod": "[variables('global_var_allocationMethod')]",
                        "publicIpAddress": "[if(equals(variables('global_fw_interface1_pip_option'),'Yes'), variables('fw2_interface1_pip_id'), json('null'))]",
                        "subnet": {
                          "id": "[concat(resourceId(variables('global_vnet_resource_group'), 'Microsoft.Network/virtualNetworks', variables('global_vnet_name')), '/subnets/', variables('global_vnet_subnet1_name'))]"
                        },
                        "primary": true,
                        "privateIPAddressVersion": "[variables('global_var_networkVersion')]",
                        "loadBalancerBackendAddressPools": [
                          {
                            "id": "[concat(resourceId(variables('publicLb_resource_group'), 'Microsoft.Network/loadBalancers', variables('publicLb_name')), '/backendAddressPools/', variables('publicLb_pool_name'))]"
                          }
                        ]
                      }
                    }
                  ],
                  "enableAcceleratedNetworking": "[variables('global_fw_enableAcceleratedNetworking')]",
                  "enableIPForwarding": true,
                  "networkSecurityGroup": {
                    "id": "[resourceId('Microsoft.Network/networkSecurityGroups', variables('global_fw_dataNsg_name'))]"
                  },
                  "tapConfigurations": []
                },
                "dependsOn": [
                  "[resourceId('Microsoft.Network/publicIPAddresses/', variables('fw2_interface1_pip_name'))]"
                ]
              },
              {
                "type": "Microsoft.Network/networkInterfaces",
                "name": "[variables('fw2_interface2_name')]",
                "apiVersion": "[variables('global_var_apiVersion')]",
                "location": "[resourceGroup().location]",
                "properties": {
                  "ipConfigurations": [
                    {
                      "name": "ipconfig1",
                      "properties": {
                        "privateIpAddress": "[variables('fw2_interface2_ip')]",
                        "privateIPAllocationMethod": "[variables('global_var_allocationMethod')]",
                        "subnet": {
                          "id": "[concat(resourceId(variables('global_vnet_resource_group'), 'Microsoft.Network/virtualNetworks', variables('global_vnet_name')), '/subnets/', variables('global_vnet_subnet2_name'))]"
                        },
                        "primary": true,
                        "privateIPAddressVersion": "[variables('global_var_networkVersion')]",
                        "loadBalancerBackendAddressPools": [
                          {
                            "id": "[concat(resourceId(variables('internalLb_resource_group'), 'Microsoft.Network/loadBalancers', variables('internalLb_name')), '/backendAddressPools/', variables('internalLb_pool_name'))]"
                          }
                        ]
                      }
                    }
                  ],
                  "enableAcceleratedNetworking": "[variables('global_fw_enableAcceleratedNetworking')]",
                  "enableIPForwarding": true,
                  "networkSecurityGroup": {
                    "id": "[resourceId('Microsoft.Network/networkSecurityGroups', variables('global_fw_dataNsg_name'))]"
                  },
                  "tapConfigurations": []
                }
              },
              {
                "type": "Microsoft.Compute/virtualMachines",
                "name": "[variables('fw2_computerName')]",
                "apiVersion": "[variables('global_var_apiVersion')]",
                "location": "[resourceGroup().location]",
                "plan": {
                  "name": "[variables('global_fw_license')]",
                  "product": "[variables('global_fw_product')]",
                  "publisher": "[variables('global_fw_publisher')]"
                },
                "properties": {
                  "availabilitySet": "[if(equals(variables('global_fw_avset_option'),'Do not use availability set'), json('null'), variables('global_fw_avset_id'))]",
                  "hardwareProfile": {
                    "vmSize": "[variables('global_fw_vmSize')]"
                  },
                  "storageProfile": {
                    "imageReference": {
                      "publisher": "[variables('global_fw_publisher')]",
                      "offer": "[variables('global_fw_product')]",
                      "sku": "[variables('global_fw_license')]",
                      "version": "[variables('global_fw_version')]"
                    },
                    "osDisk": {
                      "createOption": "FromImage",
                      "caching": "ReadWrite",
                      "managedDisk": {
                        "storageAccountType": "[variables('global_fw_storageAccountType')]"
                      },
                      "diskSizeGB": "[variables('global_fw_diskSizeGB')]"
                    }
                  },
                  "osProfile": {
                    "computerName": "[variables('fw2_computerName')]",
                    "adminUsername": "[variables('global_fw_adminUsername')]",
                    "adminPassword": "[variables('global_fw_adminPassword')]",
                    "customData": "[base64(variables('global_fw_bootstrap'))]"
                  },
                  "networkProfile": {
                    "networkInterfaces": [
                      {
                        "id": "[resourceId('Microsoft.Network/networkInterfaces', variables('fw2_interface0_name'))]",
                        "properties": {
                          "primary": true
                        }
                      },
                      {
                        "id": "[resourceId('Microsoft.Network/networkInterfaces', variables('fw2_interface1_name'))]",
                        "properties": {
                          "primary": false
                        }
                      },
                      {
                        "id": "[resourceId('Microsoft.Network/networkInterfaces', variables('fw2_interface2_name'))]",
                        "properties": {
                          "primary": false
                        }
                      }
                    ]
                  }
                },
                "dependsOn": [
                  "[resourceId('Microsoft.Network/networkInterfaces', variables('fw2_interface0_name'))]",
                  "[resourceId('Microsoft.Network/networkInterfaces', variables('fw2_interface1_name'))]",
                  "[resourceId('Microsoft.Network/networkInterfaces', variables('fw2_interface2_name'))]"
                ]
              }
            ]
          }
        }
      }
    ]
  }
  