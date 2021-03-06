# apt-transport-in-toto [![Build Status](https://travis-ci.com/in-toto/apt-transport-in-toto.svg?branch=develop)](https://travis-ci.com/in-toto/apt-transport-in-toto) [![Coverage Status](https://coveralls.io/repos/github/in-toto/apt-transport-in-toto/badge.svg?branch=develop)](https://coveralls.io/github/in-toto/apt-transport-in-toto?branch=develop)

A custom transport method for apt that verifies the reproducibility of a Debian
package before its installation. Verification is performed with
[*in-toto*](https://in-toto.io), using a supply chain definition (*in-toto layout*)
and gathering the corresponding evidence (*in-toto links*) about the reproducibility of a package
from public [*rebuilders*](https://salsa.debian.org/reproducible-builds/debian-rebuilder-setup).


### Installation
The transport method must be an executable in `/usr/lib/apt/methods/` and its
dependencies must be installed.

---
**NOTE:** *This is a temporary solution until this transport is available as
Debian package (see #11).*

---

```shell
# Get sources
git clone https://github.com/in-toto/apt-transport-in-toto.git
# Install requirements
pip install -r apt-transport-in-toto/requirements.txt
# Install transport
ln -s /usr/lib/apt/methods/intoto apt-transport-in-toto/intoto.py
chmod 755 /usr/lib/apt/methods/intoto
```


### Configuration
---
**NOTE:** *Once this transport is available as Debian package, default
configuration and installation of required metadata may be performed
automatically on installation of the package
(see [#11](https://github.com/in-toto/apt-transport-in-toto/issues/1)).*

---

#### Layout
To define the requirement of reproducibility for a package, an in-toto layout
must be available on the client at verification time and its path must be
specified in the apt configuration file (see
[*Options*](https://github.com/in-toto/apt-transport-in-toto#options) below).

A generic rebuild layout can be found in [`data/root.layout`](data/root.layout)
and may be used to verify any package. It contains public keys to verify the
authenticity and integrity of rebuilder link metadata and a threshold that
specifies how many authorized rebuilders need to agree on their result.

---
**NOTE:** *Update the layout to add or revoke rebuilder authorizations.
See discussion in [#13](https://github.com/in-toto/apt-transport-in-toto/issues/13)
for further details.*

---

#### Layout keys
For a successful verification the layout requires at least one valid signature.
The signing key(s) are the root of trust and must be available in a gpg keyring
on the client. The corresponding keyid(s) must be specified in the apt
configuration file (see
[*Options*](https://github.com/in-toto/apt-transport-in-toto#options) below).

---
**NOTE:** *Downstream maintainers should manually verify the validity of
[`data/root.layout`](data/root.layout) and sign it with their maintainer key.
See discussion in [#13](https://github.com/in-toto/apt-transport-in-toto/issues/13)
for further details.*

---

#### Options
Below options must be configured in `/etc/apt/apt.conf.d/intoto`.

- *Rebuilders* -- URIs of remote rebuilders that serve in-toto link metadata
  for package rebuilds
- *in-toto layout* -- Path to supply chain definition
- *Layout keyids* -- Keyid(s) of in-toto layout signing key(s)
- *GPGHomedir (optional)* -- Path to a non-default gpg keyring
- *LogLevel (optional)* -- Transport verbosity level during installation
  ([numeric value](https://docs.python.org/3/library/logging.html#logging-levels))
- *NoFail (optional)* -- If set to "true" installation continues after a
  verification failure, but only if the failure reason is missing link
  metadata. This option may be used for a slow roll-out. It should be disabled
  once there is broad network of rebuilders that provide extensive link
  metadata.

An exemplary configuration file can be found in
[`apt.conf.d/intoto`](apt.conf.d/intoto).

#### Enable the transport
Verification is enabled by specifying the transport method as protocol prefix
`"intoto"` in `/etc/apt/sources.list` or `/etc/apt/sources.list.d/*`, e.g.:
```
deb intoto://ftp.us.debian.org/debian/ stretch main contrib
```

### Usage
The in-toto apt transport works transparently in the background when running:

```
apt-get install <package name>
```

### Testing
The test suite can be run locally with `tox`.

#### Testing with docker
In addition to the offline Python tests that mock `apt` and `rebuilder`
behavior, there is a docker setup that installs the apt transport in a minimal
Debian container and invokes it using `apt-get install <package name>`,
fetching metadata from live rebuilders. Run the following snippet in the root
of this repo and look at the generated output.

```shell
docker build -t apt -f tests/Dockerfile .
docker run -it apt
```