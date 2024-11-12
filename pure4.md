* PURE: 4
* Title: Simplifying the Uptane Director
* Version: 1
* Last-Modified: 9-November-2024
* Author: Yashovardhan Bapat
* Status: Draft
* Content-Type: text/markdown
* Created: 08-November-2024

# PURE-4: Simplifying the Uptane Director

## Abstract

The Director in Uptane is responsible for instructing ECUs as to which images
will be installed by producing signed metadata on demand. It is mostly controlled
by automated processes. It also consults a private inventory database containing
information on vehicles, ECUs, and software revisions.

The Director in the current design uses all 4 TUF roles - the Root, Targets,
Snapshot and Timestamp. This PURE demonstrates that the Snapshot and Targets
roles and their associated metadata is redundant and proposes an alternative that
would be better received by the industry for its ease of adoption, considering
it integrates the existing Public Key Infrastructure into Uptane, while maintaining
Uptane's security posture and properties.

## Motivation

The 4 TUF roles serve purposes of generating and signing metadata about different
things. The Targets metadata (produced by the Targets role) intends to protect the
integrity of the actual update package being sent over. The Snapshot metadata
intends to protect the Targets metadata, and the Timestamp protects the Snapshot
metadata, as shown in the representation below:

```
|--------------------------------------------------------------------------------|
| Timestamp --protects--> Snapshot --protects--> Targets --protects--> [Package] |
|--------------------------------------------------------------------------------|
```

It has been observed from numerous accounts that adoption of the Uptane standard
in the industry has been met with some challenges, because of its complicated
nature. One aspect of resolving this issue was to solve redundancies and simplify
parts of the standard which can be.

The Snapshot metadata protects the Targets metadata by signing information about
all the Targets metadata files on the repository.

During the update delivery and installation cycle for a vehicle (or set of vehicles
receiving the same update), there will be only one package. This package will
contain all the images to be installed on the relevant ECUs of the car.

This package will have a Targets metadata file associated with it. However, since
there is only one package and one associated Targets metadata file, the Snapshot
metadata file associated with this update cycle will effectively have the same
information in it, making it redundant. Further, the Timestamp metadata file
associated with this "single-entity" Snapshot metadata file will also be
redundant.

The security features provided by the 3 roles (Targets, Snapshot and Timestamp)
can be analyzed by understanding the types of attacks they protect the system
against.

Broadly, these roles and their metadata protect against 4 types of attacks:
* Freeze Attack
* Rollback Attack
* Mix-and-match Attack
* Arbitrary Software Attack

### Threat Landscape Analysis

#### Freeze Attack

_Definition_: Continue to send a properly signed, but old, update bundle to the
ECUs, even if newer updates exist.

_Mitigation (in Uptane)_: Check that the latest securely attested timestamp
attestation is lower than the expiration timestamp set for the current metadata.

_Comments_: The same mitigation is used for all - Timestamp, Snapshots and
Targets metadata.

#### Rollback Attack

_Definition_: Cause an ECU to install a previously valid software revision that
is older than the currently installed version.

_Mitigation (in Uptane)_: Using a monotonically increasing metadata and image 
version number and checking that the current number is higher than the previous 
one.

_Comments_: The same mitigation is used for all - Timestamp, Snapshots and 
Targets metadata

#### Arbitrary Software Attack

_Definition_:  Cause an ECU to install and run arbitrary code of the attacker’s
choice.

_Mitigation (in Uptane)_: Checking that the metadata has been signed by the 
threshold of unique keys specified in the latest Root metadata file. 

_Comments_: The same mitigation is used for all - Timestamp, Snapshots and Targets
metadata

#### Mix-and-Match Attack

_Definition_: Install a malicious software bundle in which some of the updates do 
not interoperate properly. 

_Mitigation (in Uptane)_: Use snapshot metadata to confirm that all hashes and 
version numbers of targets metadata files.

_Comments_: In the event of only one Targets metadata file, this check is redundant.

A more detailed threat analysis can be found [here](https://github.com/uptane/threatmodel/tree/ota-stride-2)

#### General spoofing attack

_Definition_: Impersonate the identity of the Director, making the vehicle believe
that it is interacting with a legitimate OEM Director

_Mitigation (in Uptane)_: Verify identity for all roles using Root metadata


It is clear that the security features against the above outlined attacks can be 
achieved by only one role and associated metadata file - the Targets Role. The 
important aspect of all mitigations is the information about the software update
package, (the metadata). 

By eliminating the Timestamp and Snapshot roles, we are left only with two roles.
Since the scope for this is just the Director, there can be no delegations,
thereby reducing the potential complications for this change to the standard.
This can allow us to treat the overall architecture like a linear chain.

These changes will include changes both, on the OEM update server side and on the vehicle.

## Specification

### Changes to the Director

The Targets metadata MUST contain the below fields as part of the metadata:
* Filenames of all firmware/software update image files in the bundle.
* Strong cryptographic hashes of each firmware/software update image file in the 
bundle.
* Sizes of all files in the bundle
* A monotonically incrementing counter (version number)
* A secure attestation recording the time 

This information must be stored as Targets metadata according to the TUF 
specification.

The proposed change to the current architecture is integrating it with the chain
of trust model introduced by the Public Key Infrastructure.

The Root role will be a general [Trust Anchor](https://csrc.nist.gov/glossary/term/trust_anchor). 
The Root role SHALL use offline keys to generate a self-signed certificate, with 
a long validity.

The Targets role will be an intermediate or end-entity certification authority
which uses **online keys** to generate a certificate. This certificate is 
validated and signed by the Trust Anchor.
The online keys SHOULD be generated for update distribution instance.

The Targets role on the Director will generate the Targets metadata as outlined
above, and sign it with the Targets role's keys. 
The Target role's certificate MUST be sent with the metadata to the target vehicle
in order to confirm its identity. This information SHOULD be encrypted using the
target vehicle's Primary ECU's public key.

### Changes on the vehicle

All Primary ECUs and Secondary ECUs capable of performing full verification MUST have a
list of trusted/known Trust Anchors, and a way to identify them - either their public key,
their hash or some other representation, to verify the root of trust in the system.

This list of known trust anchors SHOULD be stored in a secure, tamper-proof location 
on the device to prevent any loss of integrity.

### Full verification

The full verification workflow will largely be the [same as in the current 
standard](https://uptane.org/docs/2.1.0/standard/uptane-standard#5442-full-verification), 
except for changes to points 2, 3 and 4 (4 is eliminated):

1. Load and verify the current time or the most recent securely attested time. 
2. Download and check the certificate of the Director's Targets role. Ensure that it 
is valid.
3. Verify all certificates involved in the Director's certificate chain of trust. 
The Trust Anchor's public key(s) MUST be verified with the on-device trust store.
4. Download and check the Targets metadata file from the Director repository, following 
the procedure in Section 5.4.4.6. 
5. If the Targets metadata from the Directory repository indicates that there are no new
targets that are not already currently installed, the verification process SHOULD be
stopped and considered complete. Optionally, implementors can order vehicles to check image
repo root metadata when desirable, even in the absence of an update. Otherwise, download
and check the Root metadata file from the Image repository, following the procedure in
Section 5.4.4.3. 
6. Download and check the Timestamp metadata file from the Image repository, following the procedure in Section 5.4.4.4. 
7. Check the previously downloaded Snapshot metadata file from the Image repository (if available). If the hashes and version number of that file match the hashes and version number listed in the new Timestamp metadata, the ECU SHALL skip to the last step. Otherwise, download and check the Snapshot metadata file from the Image repository, following the procedure in Section 5.4.4.5. 
8. Download and check the top-level Targets metadata file from the Image repository, following the procedure in Section 5.4.4.6. 
9. Verify that Targets metadata from the Director and Image repositories match. A Primary ECU SHALL perform this check on metadata for all images listed in the Targets metadata file from the Director repository downloaded in step 6. A Secondary ECU can elect to perform this check only on the metadata for the image it will install. (That is, the image metadata from the Director that contains the ECU identifier of the current ECU.) To check that the metadata for an image matches, complete the following procedure:
10. Locate and download a Targets metadata file from the Image repository that contains an image with exactly the same filename listed in the Director metadata, following the procedure in Section 5.4.4.7. 
11. Check that the Targets metadata from the Image repository matches the Targets metadata from the Director repository:
    1. Check that the non-custom metadata (i.e., length and hashes) of the unencrypted or encrypted image are the same in both sets of metadata. Note: the Primary is responsible for validating encrypted images and associated metadata. The target ECU (Primary or Secondary) is responsible for validating the unencrypted image and associated metadata. 
    2. Check that all SHALL match custom metadata (e.g., hardware identifier and release counter) are the same in both sets of metadata. 
    3. Check that the release counter, if one is used, in the previous Targets metadata file is less than or equal to the release counter in this Targets metadata file.
    
If any step fails, the ECU SHALL return an error code indicating the failure. If a check for a specific type of security attack fails (i.e., rollback, freeze, arbitrary software, etc.), the ECU SHOULD return an error code that indicates the type of attack.


The rest of the full verification workflow will carry on as written down in the
Uptane Standard from points 5 through 12 inclusive.

### Partial verification

The partial verification workflow will be the same as outlined in the current 
standard. 

### Updating the Certificate Trust List on full verification-capable ECUs

Updates to this Certificate Trust List can be issued through an Uptane-compliant
system. 

## Security Analysis

As mentioned in the [Motivation](#motivation), the Snapshot and Timestamp metadata
do not offer any extra security for the attacks mentioned in the Uptane threat 
model.
Analyzing all these attacks with the new model, we can clearly see that this proposal
does not detract from existing security guarantees provided by Uptane.

#### Freeze Attack
_Definition_: Continue to send a properly signed, but old, update bundle to the
ECUs, even if newer updates exist.

_Mitigation (in Uptane)_: Check that the latest securely attested timestamp 
attestation is lower than the expiration timestamp set for the Targets metadata. 

#### Rollback Attack
_Definition_: Cause an ECU to install a previously valid software revision that
is older than the currently installed version.

_Mitigation (in Uptane)_: Using a monotonically increasing metadata and image 
version number and checking that the current number is higher than the previous 
one.

#### Arbitrary Software Attack
_Definition_:  Cause an ECU to install and run arbitrary code of the attacker’s
choice.

_Mitigation (in Uptane)_: Checking that the Targets metadata has been signed by
the threshold of unique keys specified in the latest Root metadata file. 

#### Mix-and-Match Attack
_Definition_: Install a malicious software bundle in which some of the updates do 
not interoperate properly. 

_Mitigation (in Uptane)_: Use snapshot metadata to confirm that all hashes and 
version numbers of targets metadata files.

_Comments_: In the event of only one Targets metadata file, this check is redundant.

#### General spoofing attack
_Definition_: Impersonate the identity of the Director, making the vehicle believe
that it is interacting with a legitimate OEM Director

_Mitigation (in Uptane)_: Verify identity for Targets metadata and Director by
verifying Targets certificate and the chain of trust all the way to the Trust 
Anchor with the on-board Certificate Trust List.

All previously outlined security features are intact with this architectural 
change.

## Backwards Compatability

This will not be fully backwards compatible with previous versions of Uptane as
it introduces a fundamental amendment in the overall architecture of the system
and key aspects like full verification.

Changes to full verification is a moderately severe incompatibility as a very 
large portion of Uptane's security model relies on full verification. 

Clients working with PURE-4 of the Uptane standard will not be able to use the
full verification workflow outlined in the Uptane standard, and the new workflow
has been mentioned above as a way to overcome this difficulty.
    
## Copyright

This document has been placed in the public domain.
