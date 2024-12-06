* PURE: 5
* Title: Integrating Uptane with Current Infrastructure
* Version: 1
* Last-Modified: 20-November-2024
* Author: Yashovardhan Bapat
* Status: Draft
* Content-Type: text/markdown
* Created: 20-November-2024

# PURE-5: Integrating Uptane with Existing Infrastructure

## Abstract

PURE-5 is in conjunction with PURE-4: Simplifying the Uptane Director.

It outlines a way to integrate PKI into the current standard, describing
the changes required to be made on the Director and on the vehicle.

This document proposes architectural changes to the Uptane standard to address
implementation complexity and improve adoption in the industry without
compromising its security posture. The modifications focus on integrating existing
security architectures, such as Public Key Infrastructure (PKI), into the Uptane
framework.
Key changes include replacing the Root role with a Trust Anchor similar to
that defined in the Public Key Infrastructure.

## Motivation

PURE-4 mentions that it has been observed that adoption of the Uptane standard
in the industry has been met with some challenges, because of its complicated
nature. One aspect of resolving this issue was to solve redundancies - the other
is to simplify parts of the Uptane Standard by making it easier to implement while
maintaining the same security posture.

Simplifying the implementation of the Standard will encourage adoption by 
industry stakeholders, as it reduces the additional cost and effort required to
align the Director role with the TUF specification.

A solution for making the Standard easier to implement is by integrating existing
security architectures like PKI into it.

Building off of the threats associated with the Director as in PURE-4, we can
define 5 major threats that the Director protects against:
* Freeze Attack
* Rollback Attack
* Mix-and-match Attack
* Arbitrary Software Attack
* General Spoofing Attack

PURE-4 discusses the Targets, Snapshot and Timestamp roles and metadata. PURE-5
turns its attention to the Root role and metadata. 

The Root role generates metadata mapping each role to its public key and signs this
metadata using its _offline_ key. The main function of the Root metadata is to
link each role with its associated key, facilitating verification of the keys being
used to identify different roles in the Uptane system. 

This information is used to verify the Director's identity and prevent a spoofing
attack, and further verify the identity of the Targets role.

However, the Root metadata is not the only way this can be achieved. The PKI
has a way to address these security issues using certificates and the certificate
chain of trust model.

Among the five attacks listed above, the most important one from the Root role's
perspective is the General Spoofing Attack.

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

#### General Spoofing Attack

_Definition_: Impersonate the identity of the Director, making the vehicle believe
that it is interacting with a legitimate OEM Director

_Mitigation (in Uptane)_: Verify identity for all roles using Root metadata.

A more detailed threat analysis can be found [here](https://github.com/uptane/threatmodel/tree/ota-stride-2)

Assuming both Snapshot and Timestamp roles have been removed (as PURE-4 states),
we are left with only two roles (the Root and Targets), which can be thought of 
as a linear relationship. This can also be thought of as similar to the certificate
chain of trust model.

## Specification

### Changes to the Director

The Root role will be replaced by a general [Trust Anchor](https://csrc.nist.gov/glossary/term/trust_anchor). 
The Trust Anchor SHALL use offline keys to generate a self-signed certificate
with a long validity. The OEM SHALL determine whether to implement their own Trust
Anchor or rely on a publicly available Trust Anchor, such as a Root Certification 
Authority defined in the X.509 specification.

The Targets role will be replaced by an intermediate or end-entity certification
authority which uses online keys for each session to generate a digital 
certificate. This certificate SHALL contain a field mentioning the Targets role's
function and whether it is authorized to perform the action it is performing.
This is to prevent an unauthorized, but trusted entity to ship updates to a vehicle.
This certificate MUST be validated and signed by the Trust Anchor.

The Targets role will generate Targets Metadata as outlined below and sign it with
the Targets role's keys.

The Targets metadata MUST contain the following information:
* Filenames of all firmware/software update image files in the bundle,
* Strong cryptographic hashes of each firmware/software update image file in the 
bundle,
* Sizes of all files in the bundle,
* A monotonically incrementing counter (version number),
* A secure attestation recording the time.

The Targets role's certificate MUST be shipped with this targets metadata to the
target vehicle in order to verify its identity. This information SHOULD be
encrypted using the target vehicle's Primary ECU's public key.

### Changes on the vehicle

All Primary ECUs and Secondary ECUs capable of performing full verification MUST
have a list of trusted/known Trust Anchors (Trust Anchor List) and a way to verify
their identity - either their public key, its hash or some other representation.

This Trust Anchor List SHOULD be stored in a secure, tamper-proof location
on the device to prevent any loss of integrity.

This Trust Anchor List will be pre-configured during the first installation in
the vehicle, and may be updated during the vehicle's life. The method to update
the TAL is described in a section below.

### Full Verification

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

### Updating the Trust Anchor List on full verification-capable ECUs

Updates to the Trust Anchor List on ECUs capable of performing full verification
could be one of two types:
* Change in keys
* Change in Trust Anchor

The first type has a higher likelihood of occurring, especially as Uptane encourages
key rotation for the root of trust and Targets role. 
The second type may be triggered by a number of situations, but is less likely
to occur as compared to the first one. This is simply because of the complexity
involved in changing the Trust Anchor. 

#### Change in keys

Robust key rotation is a critical aspect of the security provided by TUF and
Uptane. Key rotation is particularly important in cases where a key on the root
of trust (Trust Anchor) has been compromised. The TUF Standard outlines a
method for securely migrating to a new set of trusted root keys.
If, while executing step 3 in full verification from PURE-4, the certificate in
the ECU’s Trust Anchor List (TAL) is found to be invalid, it could indicate that
the Trust Anchor keys have been rotated. In such cases, the ECU SHOULD take
appropriate steps to retrieve the updated certificate from the Director:

* The new certificate must be published to some central repository, such as
Certificate Transparency Log. 
* In the event of a certificate being determined as invalid, the Primary must
alert the Director about the invalid certificate. 
* The Director will consult a central repository of verified and known trust
anchors, which is updated upon addition of new trust anchors and update its own
Trust Anchor List. 
* The Director will then respond to the Primary with a delta update, for adding
the new certificate in its Trust Anchor List. The Director must generate
appropriate Targets metadata before shipping this update. 
* The Primary will verify the metadata for the delta update, update its TAL and
proceed with full verification step 3 onwards.

#### Change in Trust Anchor

In the situation where the OEM decides to change the Trust Anchor, this information
must be conveyed to all target vehicles in order for the update process to work
seamlessly.

The Trust Anchor change will be one of two cases: the new Trust Anchor exists in the
onboard Trust Anchor List, and the new Trust Anchor does not exist in the
onboard Trust Anchor List.

When new Trust Anchor exists in the onboard Trust Anchor List the following
process must be followed:
* As part of the regular Uptane-compliant software update workflow, a document
stating the plan to change the Trust Anchor must be directed to be processed
by the Primary ECU - the flow remains the same, with the Primary verifying the
root of trust with the current Trust Anchor. This document may contain
information about the new TA, such as its name, public key, etc. 
* During the next software update cycle, while verifying the identity of the
Trust Anchor as part of verifying the certificate chain of trust, (Step 3 in
full verification), the Primary SHALL verify the identity of the new Trust Anchor
through the document received in the previous update cycle and its own TAL.
  * If the identities do not match, the Primary SHALL drop the received contents
    (if any), notify the Director and terminate the connection.
  * If the identities match, the Primary will continue with the rest of the full
  verification workflow.

When new Trust Anchor does not exist in the onboard Trust Anchor List the
following process must be followed:
* The OEM SHALL notify the Director about the intent to introduce a new Trust
Anchor.
* The Director will consult a central repository of verified and known trust
anchors, which is updated upon addition of new trust anchors, such as
Certificate Transparency to update its Trust Anchor List. 
* As part of a regular Uptane-compliant software update workflow, the Director
will generate a document stating the plan to add a new Trust Anchor. This document
is meant for and will be processed by the Primary ECU.
* Once this document is verified and processed by the Primary, it must contact
the Director, with a confirmation for updating its Trust Anchor List,
whenever ready.
* The Director will issue another update to this Primary, with the updated
Trust Anchor List. 
* The Primary SHALL perform full verification upon receiving the updated
Trust Anchor List. Upon successful verification, it will update its own TAL and
send a success code to the Director through the Vehicle Version Manifest the next
time it is sent.
* The Director will then derive the root of trust from the new Trust Anchor.

The details of the implementation of the above flow is left to the OEM.

## Security Analysis

No changes are made to the Targets metadata, which protects important information
regarding the update package, and protects against the four attacks: Freeze, 
Rollback, Arbitrary Software and Mix-and-Match attacks. Thus, they remain protected
to the same degree.

Since the Root role protects against spoofing attacks, we may analyze it:

#### Spoofing Attack
  
_Definition_: Impersonate the identity of the Director, making the vehicle believe
that it is interacting with a legitimate OEM Director.

_Mitigation_: Verify identity and level of access for Director by verifying
Targets certificate, the access field in said certificate and the chain of trust
all the way to the Trust Anchor with the on-board Trust Anchor List.

All other types of attacks protected by other security features remain protected
with this architectural change.

#### Updating the onboard Trust Anchor List

The TAL is the method by which a Primary knows whether to trust a Trust Anchor
or not. Securing updates to the TAL is of paramount importance.

The method outlined above for updating the TAL uses a combination of Uptane's
secure update system and existing secure methods used in PKI to update TALs.
This protects the system from all attacks included in PKI and Uptane's threat
model, including some very nuanced DoS attacks (such as the ones described in
Uptane).

The specified method even protects against endless data attacks, by allowing
the Primary ECU to decide when to receive the updated TAL from the Director.

## Backwards Compatibility

This change will not be fully backwards compatible with previous versions of
Uptane as it introduces a fundamental amendment in the overall architecture of
the system (the Root role).

Changes to the Root role is a moderately severe incompatibility as a large 
portion of Uptane's security model relies on identity and access level security
provided by the Root role.

Clients working with PURE-5 of the Uptane Standard will not be able to use the
Root role outlined in the Uptane Standard, and the alternative solution has been
presented in this document.

## Copyright

This document has been placed in the public domain.
