* PURE: 4
* Title: Simplifying the Uptane Director
* Version: 1
* Last-Modified: 20-November-2024
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
roles and their associated metadata is redundant and that the director can
provide the same security services with only the Root and Targets metadata.

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
parts of the standard which can be simplified.

The Snapshot metadata protects the Targets metadata by signing information about
all the Targets metadata files on the repository.

During the update delivery and installation cycle for a vehicle (or set of vehicles
receiving the same update), there will be only one package. This package will
contain all the images to be installed on the relevant ECUs of the car.

This package will have a Targets metadata file associated with it. However, in 
this session, since there is only one package and one associated Targets 
metadata file, the Snapshot metadata file associated with this update cycle will
effectively have the same information in it, making it redundant. Further, the 
Timestamp metadata file associated with this "single-entity" Snapshot metadata 
file will also be redundant.

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

By eliminating the Timestamp and Snapshot roles, we are left only with two roles.
Since the scope for this is just the Director, there can be no delegations,
thereby reducing the potential complications for this change to the standard.

These changes will affect both, the OEM's update server (where the Director is housed)
and the vehicle.

## Specification

### Changes to the Director

The Director will have two TUF-like roles: Root and Targets.

The Root metadata distributes the public keys of the top-level Root
and Targets roles. This will remain the same as in the TUF spec.

The Targets metadata MUST contain the below fields as part of the metadata:
* Filenames of all firmware/software update image files in the bundle,
* Strong cryptographic hashes of each firmware/software update image file in the 
bundle,
* Sizes of all files in the bundle,
* A monotonically incrementing counter (version number),
* A secure attestation recording the time.

This information must be stored as Targets metadata similar to the TUF spec - 
but since this is fundamentally different to that described in TUF, we may call
it a TUF-like Targets Role and TUF-like Targets Metadata.

The proposed change to the current architecture is removing the Snapshot and Timestamp
Roles and their associated metadata from the Uptane Standard.
Their functions can be served by the single Targets Role and Metadata, provided the 
Targets metadata has all the above fields.

### Full verification

The full verification workflow will largely be the [same as in the current 
standard](https://uptane.org/docs/2.1.0/standard/uptane-standard#5442-full-verification), 
except for changes to points 3 and 4 (both are eliminated):

1. Load and verify the current time or the most recent securely attested time. 
2. Download and check the Root metadata file from the Director repository, following the procedure in Section 5.4.4.3.
3. Download and check the Targets metadata file from the Director repository, following 
the procedure in Section 5.4.4.6. 
4. If the Targets metadata from the Directory repository indicates that there are no new
targets that are not already currently installed, the verification process SHOULD be
stopped and considered complete. Optionally, implementors can order vehicles to check image
repo root metadata when desirable, even in the absence of an update. Otherwise, download
and check the Root metadata file from the Image repository, following the procedure in
Section 5.4.4.3. 
5. Download and check the Timestamp metadata file from the Image repository, following the procedure in Section 5.4.4.4. 
6. Check the previously downloaded Snapshot metadata file from the Image repository (if available). If the hashes and 
version number of that file match the hashes and version number listed in the new Timestamp metadata, the ECU SHALL skip to the last step. Otherwise, download and check the Snapshot metadata file from the Image repository, following the procedure in Section 5.4.4.5. 
7. Download and check the top-level Targets metadata file from the Image repository, following the procedure in Section 5.4.4.6. 
8. Verify that Targets metadata from the Director and Image repositories match. A Primary ECU SHALL perform this check on metadata for all images listed in the Targets metadata file from the Director repository downloaded in step 6. A Secondary ECU can elect to perform this check only on the metadata for the image it will install. (That is, the image metadata from the Director that contains the ECU identifier of the current ECU.) To check that the metadata for an image matches, complete the following procedure:
9. Locate and download a Targets metadata file from the Image repository that contains an image with exactly the same filename listed in the Director metadata, following the procedure in Section 5.4.4.7. 
10. Check that the Targets metadata from the Image repository matches the Targets metadata from the Director repository:
    1. Check that the non-custom metadata (i.e., length and hashes) of the unencrypted or encrypted image are the same in both sets of metadata. Note: the Primary is responsible for validating encrypted images and associated metadata. The target ECU (Primary or Secondary) is responsible for validating the unencrypted image and associated metadata. 
    2. Check that all SHALL match custom metadata (e.g., hardware identifier and release counter) are the same in both sets of metadata. 
    3. Check that the release counter, if one is used, in the previous Targets metadata file is less than or equal to the release counter in this Targets metadata file.
    
If any step fails, the ECU SHALL return an error code indicating the failure. If a check for a specific type of security attack fails (i.e., rollback, freeze, arbitrary software, etc.), the ECU SHOULD return an error code that indicates the type of attack.

### Partial verification

The partial verification workflow will be the same as outlined in the current 
standard. 

## Security Analysis

As mentioned in the [Motivation](#motivation), the Snapshot and Timestamp metadata
do not offer any extra security for the attacks mentioned in the Uptane threat 
model.
Analyzing all these attacks with the new architecture, we can clearly see that 
this proposal does not detract from existing security guarantees provided by Uptane.

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

All previously outlined security features are intact with this architectural 
change.

## Backwards Compatability

This will not be fully backwards compatible with previous versions of Uptane as
it introduces a fundamental amendment in the overall architecture of the system
and key aspects like full verification.

Changes to full verification is a moderately severe incompatibility as a very 
large portion of Uptane's security model relies on full verification. 

Clients working with PURE-4 of the Uptane standard will not be able to use the
full verification workflow outlined in the Uptane standard, and the modifications 
to this workflow have been mentioned above as a way to overcome this difficulty.
    
## Copyright

This document has been placed in the public domain.
