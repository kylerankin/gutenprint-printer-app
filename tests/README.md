Run the device-selection regression against a freshly built application with
its Gutenprint PPD generator installed and `PAPPL_MAX_VENDOR >= 256`:

```sh
python3 tests/device-selection.py /path/to/gutenprint-printer-app
```

For an OCI image built from the current checkout, pass the container command
(`just verify` runs this against the image it builds):

```sh
python3 tests/device-selection.py -- podman run --rm \
  --entrypoint /usr/bin/gutenprint-printer-app ghcr.io/projectbluefin/gutenprint-printer-app:build
```

The test calls the real `drivers -o device-id=...` auto-add path. Unknown PCL
IDs must return no driver, while Epson Stylus Photo R300 and Canon PIXMA iP4000
must select their registered Gutenprint model drivers, including when their IDs
advertise PCL. It requires no hardware or generic PCL payload. It fails on an
empty inventory, missing models, explicitly simplified drivers, or application errors.
PAPPL-retrofit strips the Gutenprint suffix from display names; therefore this
test alone cannot prove expert PPD provenance. Run it on the expert-only build
and retain the expert-option validation from #9.

This verifies driver selection only. It does not replace the full OCI
print-to-socket-sink verification of filter output required by the issue, or
physical paper testing. The expert-option and OCI integration work is tracked
in #9 and #4 respectively.

`tests/coexistence.sh` runs two instances of the built image on one host
(ports `PORT` and `PORT+1`, default 18400/18401, sinks at `+1000`) with
separate volumes, gives each a different synthetic Gutenprint queue, prints
from both into separate socket sinks, and checks with `tests/mdns-browse.py`
that every `_ipp._tcp`/`_ipps._tcp` advertisement on the link belongs to
exactly one instance with that instance's port and `rp=` path, before and
after recreating both containers on their volumes. `tests/mdns-browse.py
<service type> [seconds]` is a dependency-free mDNS browser (multicast
PTR/SRV/TXT queries from port 5353, legacy unicast if the port cannot be
shared) that prints one JSON record per resolved service instance; it needs
host networking for the instances and no Avahi client. Neither proves USB
or paper output.

`tests/service-advertisements.sh` is the host-network counterpart. On an
otherwise quiet test LAN it browses the real `_ssh._tcp`, `_sftp-ssh._tcp`
and `_ipp._tcp` records before and after starting and restarting two
instances with distinct names, ports and state volumes, proving the built
image ships no inherited Avahi SSH/SFTP record while each instance's own IPP
queue still resolves on its distinct port and `rp=` path. It needs host
`avahi-daemon` and `avahi-browse`, so it runs as an operator recipe
(`just verify-service-advertisements`) rather than part of `just verify`.
`tests/service-advertisements.sh` accepts `IMAGE=<ref>` to observe another
build, `PORT=<base-port>` to move the two instances off the default
`18546`/`18547`, and `EVIDENCE_DIR=<dir>` to keep the browsed records,
container logs and image metadata instead of a temporary directory. It proves
nothing about physical discovery or printed paper.
