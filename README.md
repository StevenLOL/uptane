# uptanedemo
Early demonstration code for UPTANE. Python 3 is preferred during development.

# Instructions on use of the uptane demo code
## Installation
(As usual, virtual environments are recommended for development and testing, but not necessary.)

To download and install the Uptane code, run the following:
```shell
git clone https://github.com/uptane/uptane
cd uptane
pip install cffi==1.7.0 pycrypto==2.6.1 pynacl==1.0.1 cryptography
pip install git+git://github.com/awwad/tuf.git@pinning
pip install -e .
```

If you're going to be running the ASN.1 encoding scripts once they are ready, you'll also need to `pip install pyasn1`


## Running
The code below is intended to be run IN FIVE PANES:
- WINDOW 1: Python shell for the OEM. This serves HTTP (repository files).
- WINDOW 2: Python shell for the Director (Repository and Service). This serves metadata and image files via HTTP receives manifests from the Primary via XMLRPC (manifests).
- WINDOW 3: Bash shell for the Timeserver. This serves signed times in response to requests from the Primary via XMLRPC.
- WINDOW 4: Python shell for a Primary client in the vehicle. This fetches images and metadata from the repositories via HTTP, and communicates with the Director service, Timeserver, and any Secondaries via XMLRPC.
- WINDOW 5: (At least one) Python shell for a Secondary in the vehicle. This communicates directly only with the Primary via XMLRPC, and will perform full metadata verification.


###*WINDOW 1: the OEM Repository*
These instructions start a demonstration version of an OEM's main repository
for software, hosting images and the metadata Uptane requires.

```python
import demo.demo_oem_repo as do
do.clean_slate()
do.write_to_live()
do.host()
```
After the demo, to end hosting:
```python
do.kill_server()
```


###*WINDOW 2: the Director*
The following starts a Director server, which generates metadata for specific
vehicles indicating which ECUs should install what firmware (validated against
and obtained from the OEM's main repository). It also receives and validates
Vehicle Manifests from Primaries, and the ECU Manifests from Secondaries
within the Vehicle Manifests, which capture trustworthy information about what
software is running on the ECUs, along with signed reports of any attacks
observed by those ECUs.

```python
import demo.demo_director as dd
dd.clean_slate()
dd.write_to_live()
dd.host()
dd.listen()
```

After that, proceed to the following Windows to prepare clients.
Once those are ready,  you can perform a variety of modifications / attacks.

For example, to try to have the director list a file not validated by the oem:
```python
new_target_fname = dd.demo.DIRECTOR_REPO_TARGETS_DIR + '/file5.txt'
open(new_target_fname, 'w').write('Director-created target')
dd.repo.targets.add_target(new_target_fname)
dd.write_to_live()
```

After the demo, to end HTTP hosting (but not XMLRPC serving, which requires
exiting the shell), do this (or else you'll have a zombie Python process to kill)
```python
dd.kill_server()
```


###*WINDOW 3: the Timeserver:*
The following starts a simple Timeserver, which receives requests for signed
times, bundled by the Primary, and produces a signed attestation that includes
the nonces each Secondary ECU sent the Primary to include along with the
time request, so that each ECU can better establish that it is not being tricked
into accepting a false time.
```shell
#!/bin/bash
python demo/demo_timeserver.py
```

###*WINDOW 4: the Primary client:*
(ONLY AFTER THE OTHERS HAVE FINISHED STARTING UP AND ARE HOSTING)
The Primary client started below is likely to run on a more capable and
connected ECU in the vehicle - potentially the head unit / infotainment. It will
obtain metadata and images from the OEM Repository as instructed by the Director
and distribute them appropriately to other, Secondary ECUs in the vehicle,
and it will receive ECU Manifests indicating the software on each Secondary ECU,
and bundle these into a Vehicle Manifest which it will send to the Director.
```python
import demo.demo_primary as dp
dp.clean_slate() # also listens, xmlrpc
```
AFTER at least one Secondary client has been set up and submitted
(next window's section), you can try out normal operation:
```python
dp.generate_signed_vehicle_manifest()
dp.submit_vehicle_manifest_to_director()
```


###*WINDOW 5+: the Secondary client(s):*
(ONLY AFTER THE OTHERS HAVE FINISHED STARTING UP AND ARE HOSTING)
Here, we start a single Secondary ECU and generate a signed ECU Manifest
with information about the "firmware" that it is running, which we send to the
Primary.
```python
import demo.demo_secondary as ds
ds.clean_slate()
ds.listen()
ds.generate_signed_ecu_manifest()   # saved as ds.most_recent_signed_manifest
ds.submit_ecu_manifest_to_primary() # optionally takes different signed manifest
```

At this point, some attacks can be performed here, such as:

####Attack: MITM without Secondary's key changes ECU manifest:
The next command simulates a Man in the Middle attack that attempts to modify
the ECU Manifest from this Secondary to issue a false report. In this simple
attack, we simulate the Man in the Middle modifying the ECU Manifest without
modifying the signature (which, without the ECU's private key, it cannot
produce).
```python
# Send the Primary a signed ECU manifest that has been modified after the signature.
ds.ATTACK_send_corrupt_manifest_to_primary()
```
Then run this in Window 4 (the Primary's window), to cause the Primary to bundle
this ECU Manifest in its Vehicle Manifest and send it off to the Director:
```python
dp.generate_signed_vehicle_manifest()
dp.submit_vehicle_manifest_to_director()
```
As you can now see from the output in Window 2, the Director discards the bad
ECU Manifest from this Secondary, and retains the rest of the Vehicle Manifest
from the Primary.

####Attack: MITM modifies ECU manifest and signs with another ECU's key:
The next command also simulates a Man in the Middle attack that attempts to modify
the ECU Manifest from this Secondary to issue a false report. In this attack,
we simulate the Man in the Middle modifying the ECU Manifest and replacing the
signature with a valid signature from a *different* key (obtained, say, from
another vehicle's analogous ECU after reverse-engineering).
```python
ds.ATTACK_send_manifest_with_wrong_sig_to_primary()
```
Then run this in Window 4 (the Primary's window):
```python
dp.generate_signed_vehicle_manifest()
dp.submit_vehicle_manifest_to_director()
```
As you can now see from the output in Window 2, the Director discards the bad
ECU Manifest from this Secondary, and retains the rest of the Vehicle Manifest
from the Primary.