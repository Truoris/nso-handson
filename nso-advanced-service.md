
# NSO Advanced Loopback Service

We will edit the YANG model from the previous Enhanced Loopback Service with the following enhancements:

- configure a list of addresses to be configured for the loopback interface, a maximum of one IPv4 address is accepted when multiple IPv6 addresses are, at least one address has to be configured

This time, a business logic written in Python is used to distinguish between IPv4 and IPv6 addresses entered as list of addresses in the input parameters. The Python logic will also compute the netmask and prefix to be extracted from the IPv4 address and it will limit the number of IPv4 addresses to a maximum of one per loopback interface.

The XML configuration template is updated to allow both variables defined in the YANG model and in the Python code.

- Examine the YANG service model and update your loopback service
```yang
module loopback-service {
    namespace "http://com/example/loopbackservice";
    prefix loopback-service;
    
    import ietf-inet-types {
        prefix inet;
    }
    import tailf-ncs {
        prefix ncs;
    }
    
    revision 2021-05-03 {
        description "advanced version";
    }
    
    revision 2021-05-01 {
        description "enhanced version";
    }
    
    revision 2021-04-23 {
        description "Initial revision.";
    }

    typedef ip-address {
        type union {
            type inet:ipv4-prefix;
            type inet:ipv6-address;
        }
    }

    list loopback-service {
        key name;
        
        uses ncs:service-data;
        ncs:servicepoint "loopback-service-servicepoint";
        
        leaf name {
            type string {
                pattern "[a-zA-Z0-9\\-_]+";
            }
        }
        
        leaf device {
            type leafref {
                path "/ncs:devices/ncs:device/ncs:name";
            }
            mandatory true;
        }
        
        leaf id {
            type uint16{
                range "0..100";
            }
            default "100";
        }
        
        leaf-list address {
            type ip-address;
            min-elements 1;
        }
    }
}
```

- Examine the XML configuration template and update your loopback service

```xml
<config-template xmlns="http://tail-f.com/ns/config/1.0">
  <devices xmlns="http://tail-f.com/ns/ncs">
    <device>
      <name>{/device}</name>
      <config>
        <interface xmlns="urn:ios">
          <Loopback>
            <name>{/id}</name>
            <ip when="{$IPV4_ADDR !=''}">
              <address>
                <primary>
                  <address>{$IPV4_ADDR}</address>
                  <mask>{$NETMASK}</mask>
                </primary>
              </address>
            </ip>
            <ipv6 when="{$IPV6_ADDR !=''}">
              <address>
                <prefix-list>
                  <prefix>{$IPV6_ADDR}</prefix>
                </prefix-list>
              </address>
            </ipv6>
          </Loopback>
        </interface>
      </config>
    </device>
  </devices>
</config-template>
```

- Examine the Python code: ncs-run/packages/loopback-service/python/loopback_service/main.py

```python
# -*- mode: python; python-indent: 4 -*-
import ncs
from ncs.application import Service
import ipaddress


# ------------------------
# SERVICE CALLBACK EXAMPLE
# ------------------------
class ServiceCallbacks(Service):

    # The create() callback is invoked inside NCS FASTMAP and
    # must always exist.
    @Service.create
    def cb_create(self, tctx, root, service, proplist):
        self.log.info('Service create(service=', service._path, ')')


        loopback_service_template_vars = ncs.template.Variables()
        loopback_service_template_template = ncs.template.Template(service)


        ipv4_count = 0
        for address in service.address:

            ip_addr = ipaddress.ip_interface(address)

            if ip_addr.version == 4:
                ipv4_count += 1
                if ipv4_count > 1:
                    raise Exception("Only one ipv4 address can be configured")
                
                loopback_service_template_vars.add('IPV4_ADDR', ip_addr.ip)
                loopback_service_template_vars.add('NETMASK', ip_addr.netmask)
                loopback_service_template_vars.add('IPV6_ADDR', '')

            else:
                loopback_service_template_vars.add('IPV4_ADDR', '')
                loopback_service_template_vars.add('NETMASK', '')
                loopback_service_template_vars.add('IPV6_ADDR', ip_addr.ip)

            loopback_service_template_template.apply('loopback-service-template', loopback_service_template_vars)


# ---------------------------------------------
# COMPONENT THREAD THAT WILL BE STARTED BY NCS.
# ---------------------------------------------
class Main(ncs.application.Application):
    def setup(self):
        # The application class sets up logging for us. It is accessible
        # through 'self.log' and is a ncs.log.Log instance.
        self.log.info('Main RUNNING')

        # Service callbacks require a registration for a 'service point',
        # as specified in the corresponding data model.
        #
        self.register_service('loopback-service-servicepoint', ServiceCallbacks)

        # If we registered any callback(s) above, the Application class
        # took care of creating a daemon (related to the service/action point).

        # When this setup method is finished, all registrations are
        # considered done and the application is 'started'.

    def teardown(self):
        # When the application is finished (which would happen if NCS went
        # down, packages were reloaded or some error occurred) this teardown
        # method will be called.

        self.log.info('Main FINISHED')
```
  
- Make sure no loopback services are configured on NSO instance

```console
admin@ncs(config)# no loopback-service
admin@ncs(config)# commit
% No modifications to commit.
```

- Compile the package

```bash
cisco@ubuntu:~/ncs-run/packages/loopback-service/src$ make
mkdir -p ../load-dir
mkdir -p java/src//
/home/cisco/ncs-5.5/bin/ncsc  `ls loopback-service-ann.yang  > /dev/null 2>&1 && echo "-a loopback-service-ann.yang"` \
              -c -o ../load-dir/loopback-service.fxs yang/loopback-service.yang
```

- Log into NSO CLI and reload the packages

```console
cisco@ubuntu:~/ncs-run$ ncs_cli -u admin -C

User admin last logged in 2021-05-03T16:40:49.959944-04:00, to ubuntu, from 10.16.11.1 using cli-ssh
admin connected from 10.16.11.1 using ssh on ubuntu
admin@ncs# packages reload

>>> System upgrade is starting.
>>> Sessions in configure mode must exit to operational mode.
>>> No configuration changes can be performed until upgrade has completed.
>>> System upgrade has completed successfully.
reload-result {
    package cisco-ios-cli-6.85
    result true
}
reload-result {
    package loopback-service
    result true
}
admin@ncs# 
System message at 2021-05-03 16:43:26...
    Subsystem stopped: ncs-dp-5-cisco-ios-cli-6.85:IOSDp
admin@ncs# 
System message at 2021-05-03 16:43:26...
    Subsystem started: ncs-dp-6-cisco-ios-cli-6.85:IOSDp
```

- Configure a loopback service

```console
admin@ncs(config)# loopback-service loo-1 id 1 device simCPE0 address [ 10.10.10.10/32 ]
admin@ncs(config-loopback-service-loo-1)# commit dry-run
cli {
    local-node {
        data  devices {
                  device simCPE0 {
                      config {
                          interface {
             +                Loopback 1 {
             +                    ip {
             +                        address {
             +                            primary {
             +                                address 10.10.10.10;
             +                                mask 255.255.255.255;
             +                            }
             +                        }
             +                    }
             +                }
                          }
                      }
                  }
              }
             +loopback-service loo-1 {
             +    device simCPE0;
             +    id 1;
             +    address [ 10.10.10.10/32 ];
             +}
    }
}
admin@ncs(config-loopback-service-loo-1)# commit
Commit complete.
admin@ncs(config-loopback-service-loo-1)# end
```

- Observe the service configuration

```console
admin@ncs# show running-config loopback-service loo-1 
loopback-service loo-1
 device  simCPE0
 id      1
 address [ 10.10.10.10/32 ]
!
```

- Add an IPv6 address to the loopback interface

```console
admin@ncs# config t
Entering configuration mode terminal
admin@ncs(config)# loopback-service loo-1 address [ 10.10.10.10/32 2001::1 ]
admin@ncs(config-loopback-service-loo-1)# commit dry-run
cli {
    local-node {
        data  devices {
                  device simCPE0 {
                      config {
                          interface {
                              Loopback 1 {
                                  ipv6 {
                                      address {
             +                            prefix-list 2001::1 {
             +                            }
                                      }
                                  }
                              }
                          }
                      }
                  }
              }
              loopback-service loo-1 {
             -    address [ 10.10.10.10/32 ];
             +    address [ 2001::1 10.10.10.10/32 ];
              }
    }
}
admin@ncs(config-loopback-service-loo-1)# commit 
Commit complete.
```
