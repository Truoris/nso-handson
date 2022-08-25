# NSO Enhanced Loopback Service

We will edit the YANG model from the previous Base Loopback Service with the following enhancements:

- add an `id` for the loopback interface within 0..100 range and default value set to 100
- define an `IPv4 address` and a `netmask` as well as an `IPv6 address` for the loopback interface address configuration
- replace the reference to the `device` to be configured with only one device and not a list of devices
- make sure the `name` of the service of type string is compliant with a given pattern

The XML configuration template is updated to allow these new variables defined in the YANG model.

- Examine the updated YANG service model and update your loopback service
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
    description "enhanced version";
  }

  revision 2021-04-23 {
    description "Initial revision.";
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

    container ipv4 {
      presence "ipv4";
      leaf address {
        type inet:ipv4-address;
      }

      leaf netmask {
         type inet:ipv4-address;
      }
    }

    container ipv6 {
      presence "ipv6";
      leaf address {
        type inet:ipv6-address;
      }
    }

    must "ipv4 or ipv6" {
        error-message "Please assign value for at least one of the fields : ipv6/address, ipv4/address";
    }
  }

}
```

- Examine the updated XML configuration template and update your loopback service
```xml
<config-template xmlns="http://tail-f.com/ns/config/1.0">
  <devices xmlns="http://tail-f.com/ns/ncs">
    <device>
      <name>{/device}</name>
      <config>
        <interface xmlns="urn:ios">
          <Loopback>
            <name>{/id}</name>
            <ip when="{/ipv4}">
              <address>
                <primary>
                  <address>{/ipv4/address}</address>
                  <mask>{/ipv4/netmask}</mask>
                </primary>
              </address>
            </ip>
            <ipv6 when="{/ipv6}">
              <address>
                <prefix-list>
                  <prefix>{/ipv6/address}</prefix>
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

- Make sure no loopback services are configured on NSO instance

```console
admin@ncs(config)# no loopback-service *
admin@ncs(config)# commit
% No modifications to commit.
```

- Compile the service package

```bash
cisco@ubuntu:~/ncs-run/packages/loopback-service/src$ make
mkdir -p ../load-dir
/home/cisco/ncs-5.5/bin/ncsc  `ls loopback-service-ann.yang  > /dev/null 2>&1 && echo "-a loopback-service-ann.yang"` \
              -c -o ../load-dir/loopback-service.fxs yang/loopback-service.yang
```

- Reload NSO packages

```console
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
System message at 2021-05-04 02:37:48...
    Subsystem stopped: ncs-dp-8-cisco-ios-cli-6.85:IOSDp
admin@ncs# 
System message at 2021-05-04 02:37:48...
    Subsystem started: ncs-dp-9-cisco-ios-cli-6.85:IOSDp
```

- Configure a loopback service
  
```console
admin@ncs(config)# loopback-service loo-1 id 1 device simCPE0 ipv4 address 10.10.10.10 netmask 255.255.255.255
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
             +    ipv4 {
             +        address 10.10.10.10;
             +        netmask 255.255.255.255;
             +    }
             +}
    }
}
admin@ncs(config-loopback-service-loo-1)# commit
Commit complete.
admin@ncs(config-loopback-service-loo-1)# end
admin@ncs#
```

- Observe the service configuration

```console
admin@ncs# show running-config loopback-service loo-1
loopback-service loo-1
 device simCPE0
 id     1
 ipv4 address 10.10.10.10
 ipv4 netmask 255.255.255.255
!
```

- Add an IPv6 address to the loopback interface

```console
admin@ncs# config
Entering configuration mode terminal
admin@ncs(config)# loopback-service loo-1 ipv6 address 2001::1
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
             +    ipv6 {
             +        address 2001::1;
             +    }
              }
    }
}
admin@ncs(config-loopback-service-loo-1)# commit
Commit complete.
admin@ncs(config-loopback-service-loo-1)# end
```

- Observe the service configuration

```console
admin@ncs# show loopback-service loo-1 
loopback-service loo-1
 modified devices [ simCPE0 ]
 directly-modified devices [ simCPE0 ]
 device-list   [ simCPE0 ]
 plan-location /loopback-service[name='loo-1']
```

- Observe the device configuration

```console
ubuntu@nso:~/ncs-run$ ncs-netsim cli-c simCPE0
simCPE0# show running-config loopback-service loo-1
interface Loopback1
 no shutdown
 ip address 10.10.10.10 255.255.255.255
 ipv6 address 2001::1
exit
```

## Advanced Service

You can now move to the last step, the [advanced service](nso-advanced-service.md).