## Troubleshooting with host OS access

Host OS SSH access gives you a handful of tools that can help you gather more information about potential issues on your device.

__Warning:__ Making changes to running services and network configurations carries the risk of losing access to your device. Before making changes to the host OS of a remote device, it is best to test locally. Changes made to the host OS will not be maintained when the OS is updated, and some changes could break the updating process. When in doubt, [reach out][forums] to us for guidance.

### {{ $names.os.upper }} services

balenaOS uses **[systemd][systemd]** as its init system, and as such, almost all the fundamental components in balenaOS run as systemd services. In general, some core services need to execute for a device to come online, connect to the balenaCloud VPN, download applications, and then run them:

* `chronyd.service` - Responsible for NTP duties and syncing ‘real’ network time to the device.
* `dnsmasq.service` - The local DNS service which is used for all host OS lookups.
* `NetworkManager.service` - The underlying Network Manager service, ensuring that configured connections are used for networking.
* `os-config.service` - Retrieves settings and configs from the API endpoint, including certificates, authorized keys, the VPN config, etc.
* `openvpn.service` - The VPN service itself, which connects to the balenaCloud VPN, allowing a device to come online.
* `balena.service` - The [balenaEngine][balena-engine] service, the modified [Docker][docker] daemon fork that allows the management and running of application service images, containers, volumes, and networking.
* `resin-supervisor.service` - The balena Supervisor service, responsible for the management of applications, including downloading updates for and self-healing (via monitoring) of those applications, variables (application/device/fleet), and exposure of these services to applications via an endpoint.
* `dbus.service` - The DBus daemon socket which can be used by applications by applying the `io.balena.features.dbus` [label][labels], which exposes it in-container. This allows applications to control several host OS features, including the Network Manager.

Additionally, there are a couple of utility services that, while not required for a barebones operation, are also useful:

* `ModemManager.service` - Deals with non-Ethernet or Wifi devices, such as LTE/GSM modems.
* `avahi-daemon.service` - Used to broadcast the device’s local hostname.

You may see all enabled services on the host OS with the following command:

```shell
$ systemctl list-unit-files | grep enabled
```

To check the status of a service, use the `systemctl status <serviceName>` command. The output includes whether the service is currently loaded and active, together with detail about the process, including the latest entries from the journal log.  For example, to obtain the status of the OpenVPN service use the following command:

```shell
$ systemctl status openvpn.service
```

### Checking logs

#### journalctl

Information from a variety of services can be found using the **journalctl** utility. The output from journalctl can be very large and typically the output would be filtered by using the `--unit` (or the short version `-u`) option to output logs from a single service.

A typical example of using journalctl might be following a service to see what’s occuring in real time by using the `--follow` or `-f` option.

```shell
$ journalctl --follow --unit resin-supervisor
```

To limit the output to the last *x* messages, use `-n x`:

```shell
$ journalctl -n 10 -u chronyd
```

The --all or `-a` option may be used to show all entries, even if long or with unprintable characters. This is especially useful for displaying the service container logs from applications when applied to balena.service.

```shell
$ journalctl --all -n 100 -u balena
```

#### dmesg

For displaying messages from the kernel, you can use **dmesg**. Similar to **journalctl**, **dmesg** will have a very large output without some additional options:

The following example limits the output to the last 100 lines.

```shell
$ dmesg | tail -n 100
```

### Monitor {{ $names.engine.lower }}

{{ $names.os.upper }}, beginning with version 2.9.0, includes the lightweight container engine **[{{ $names.engine.lower }}][engine-link]** to manage **Docker** containers. If you think the supervisor or application container may be having problems, you’ll want to use **balena** for debugging.

From the host OS this command will show the status of all containers:

```shell
$ balena ps -a
```

You can also check the **journalctl** logs for messages related to **{{ $names.company.lower }}**:

```
journalctl --follow -n 100 -u {{ $names.engine.lower }}
```

__Note:__ For devices with {{ $names.os.lower }} versions earlier than 2.9.0, you can replace `balena` in these commands with `docker`.

### Inspect network settings

#### NetworkManager

**NetworkManager** includes a [CLI][nmcli] that can be useful for debugging your ethernet and WiFi connections. The `nmcli` command, on its own, will show all interfaces and the connections they have. `nmcli c` provides a connection summary, showing all known connection files with the connected ones highlighted. `nmcli d` displays all network interfaces (devices).

Another useful place to look for **NetworkManager** information is in the **journalctl** logs:

```shell
$ journalctl -f -n 100 -u NetworkManager
```

#### ModemManager

Similar to **NetworkManager**, **ModemManager** includes a [CLI][mmcli], `mmcli`, to manage cellular connections. `mmcli -L` provides a list of available modems.

### Look up version information

Knowing what version of a specific service is being run on your device can help you troubleshoot compatibility issues, known bugs, and supported features.

Many services provide a direct option for displaying their version:

```shell
$ udevadm --version
$ systemctl --version
$ openssl version
```

### Understand the file system

In some cases, you may need to examine the contents of certain directories or files directly. One location that is useful for troubleshooting purposes is the `/data` directory, which contains your device's Docker images, [persistent application data][persistent-storage], and host OS update logs. The `boot` directory includes configuration files, such as [config.json][config-json], [config.txt][config-txt] and [**NetworkManager** connections][network].

Note that the [filesystem layout][filesystem] may look slightly different from what you’d expect—for example, the two locations mentioned above are found at `/mnt/data` and `/mnt/boot`, respectively.

[forums]:{{$names.forums_domain}}/c/balena-cloud
[engine-link]:{{ $links.engineSiteUrl }}
[nmcli]:https://fedoraproject.org/wiki/Networking/CLI
[mmcli]:https://www.freedesktop.org/software/ModemManager/man/1.8.0/mmcli.8.html
[persistent-storage]:/learn/develop/runtime/#persistent-storage
[config-txt]:/reference/OS/advanced/#config-txt
[network]:/reference/OS/network/2.x
[filesystem]:/reference/OS/overview/2.x/#stateless-and-read-only-rootfs
[systemd]:https://www.freedesktop.org/wiki/Software/systemd/
[labels]:/learn/develop/multicontainer/#labels
[config.json]:{{ $links.osSiteUrl }}/meta-balena#configjson
