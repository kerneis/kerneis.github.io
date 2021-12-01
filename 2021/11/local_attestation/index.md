# TPM remote attestation over Bluetooth


How can you be sure that the software running on your computer has not been tampered with since you installed it?
Even if your disk is encrypted, how can you be sure that the password prompt at boot is not a trap planted by an evil third-party that will exfiltrate all your data?
Those are some of the questions that the field of "secure computing" tries to address.
The answers are built through complex combinations of hardware ([CPU](https://en.wikipedia.org/wiki/Central_processing_unit), [TPM](https://en.wikipedia.org/wiki/Trusted_Platform_Module), flash memory) and software ([BIOS](https://en.wikipedia.org/wiki/BIOS), [OS](https://en.wikipedia.org/wiki/Operating_system)) components.

In this post, I will focus on a smaller part of the problem: measured boot and remote attestation.
How do you prove to a remote third-party that your computer is the one it has on record, and that it runs the sofware it expects?
This third-party may be your network administrator, who wants to deny intruder or compromised machines on the intranet.
Or it could be yourself facing your computer: what if you could use a trusted device (eg. your phone) to communicate with your machine over bluetooth to make sure it has not been swaped or altered in your hotel room while you were enjoying a nice dinner with other conference attendees?
I recently got interested in the latter use-case, so I started researching existing standards and solutions.
*Spoiler:* I may need to build this app myself.

What follows is my understanding of the current landscape for remote attestation, from a practical perspective:
what standards exist, what implementations are available, and what remains to be done to enable local verification of measured boot over bluetooth.
I will only address Linux solutions; you can read more about Windows secure boot on [Igor's blog](https://igor-blue.github.io/2021/02/04/secure-boot.html).

> **Background**: if you never used a TPM or need a refresher on how it works, I cannot recommend enough the [TPM-JS tutorial](https://google.github.io/tpm-js) which simulates a TPM in your browser.
> The section on [remote attestation](https://google.github.io/tpm-js/#pg_attestation) is of course especially relevant here, but it's worth starting from the very begining.
> If you prefer a video, watch [Matthew Garret at LCA 2020](https://www.youtube.com/watch?v=FobfM9S9xSI).

## Existing standards

### Trusted Computing Group (TCG)

The [Trusted Computing Group](https://trustedcomputinggroup.org/) is an organisation publishing open standards and specifications about secure computing.

Specifications from the TCG provide the basic building blocks to communicate with a TPM.
For instance, the [TPM library](https://trustedcomputinggroup.org/resource/tpm-library-specification/) details the architecture of the TPM, the primitives it implements, and commands to interact with it.
For our use-case of measured boot, the [PC Client Platform Firmware Integrity Measurement](https://trustedcomputinggroup.org/resource/tcg-pc-client-platform-firmware-integrity-measurement/)
describes how the firmware should keep an event log of what software and cryptographic material is used at various stages of the boot, and attest them using the TPM.
The encoding used to represent attestation messages and the interactions between an attester (eg. your computer) and a verifier (eg. your phone) are defined in the [Trusted Attestation Protocol (TAP) Information Model](https://trustedcomputinggroup.org/resource/tcg-tap-information-model/).

However, the Trusted Attestation Protocol only defines the format of the payloads exchanged between attester and verifier.
Likewise, the PC Client FIM specification says nothing of the communication beyond an non-normative comment about
"proper selection of transmission protocols [to] maximize interoperability, freshness, and efficiency".
The choice of the transport and application layers is left unspecified by documents from the TCG.


### Remote Attestation Procedures (RATS)

The [IETF RATS Working Group](https://datatracker.ietf.org/wg/rats/about/) seeks to standardize formats for remote attestations.
Its charter is broad, ranging from FIDO to TPM and Android Keystore, but for now it seems very focused on TPM-related applications.
Since the role of the IETF is to standardize network protocols, this sounds perfect for our interest.

IETF working groups start with draft standards proposed by individual authors.
Those drafts are then adopted by the WG to be improved collectively,
and finally (after many stages of reviews) accepted as official RFC documents.
I have only reviewed the drafts adopted by RATS, because finding unadopted drafts is more tedious[^unadopted];
none of them have reached the stage of a final RFC yet.

[^unadopted]: They are not listed from the WG page, so you need to review the mailing-list archives and meeting notes to find discussions of them.

The [Architecture draft](https://tools.ietf.org/html/draft-ietf-rats-architecture) sets some definitions and goals, a framework of common terminology used in the other drafts.
The [Reference Interaction Models draft](https://datatracker.ietf.org/doc/html/draft-ietf-rats-reference-interaction-models) defines three approaches to communicate between an attester and a verifier:
challenge/response, uni-directional, and streaming.
Challenge/response remote attestation, also known as CHARRA, uses a nonce sent from the verifier to the attester to ensure freshness and avoid replay attacks.
Uni-directional and streaming rely on synchronised clocks instead.

Both of those drafts are worded in very general terms and do not mandate specific network protocols.
The [Appendix A of RIM](https://datatracker.ietf.org/doc/html/draft-ietf-rats-reference-interaction-models#appendix-A) sketches a
[CDDL](https://datatracker.ietf.org/doc/html/rfc8610) specification for a simple [CoAP](http://coap.technology/) Challenge/Response Interaction,
but it's only an informative appendix due to be deleted from the final specification.
An implementation of this protocol is [available as a proof-of-concept](https://github.com/Fraunhofer-SIT/charra).

RATS has adopted two more drafts: [TPM-based network device attestation](https://datatracker.ietf.org/doc/html/draft-ietf-rats-tpm-based-network-device-attest)
and its [YANG counterpart](https://datatracker.ietf.org/doc/html/draft-ietf-rats-yang-tpm-charra/).
Unlike the previous drafts, those two go into much more details about defining an actual, interroperable network protocol.
Unfortunately, they are not designed for the use-case at hand: the attester is assumed to be (a fleet of) network devices, such as switches, routers and servers.
They are built on top of Netconf, or its modern alternative Restconf, which are widely used by network operators.
There doesn't seem to be a prototype implementation available.

For now we are therefore left with a choice between a full-featured solution over-engineered for our use-case,
and a reasonable technological stack (CoAP) with a prototype implementation but no standardization effort.
The situation is not as grim as it may seem though: IETF working groups are opened to contribution,
so if we could build upon this prototype in an interoperable way,
it may then be possible to propose a draft for adoption to RATS.

### Constrained Application Protocol (CoAP) <a id="coap"></a>

The [Constrained Application Protocol (CoAP)](http://coap.technology/) is a specialized web transfer protocol for use with constrained nodes,
defined in [RFC 7252](https://datatracker.ietf.org/doc/html/rfc7252/).
By default, it runs over UDP or DTLS but is designed to be lightweight and portable to other transport layers such as TCP or SMS.
It also offers a flexible data model.
The [CHARRA prototype](https://github.com/Fraunhofer-SIT/charra) uses CoAP over DTLS, and [CBOR](http://cbor.io/) ([RFC 8949](https://datatracker.ietf.org/doc/html/rfc8949/)) to encode the payload.

If we want to port the proposed protocol to [Bluetooth Low Energy (BLE)](https://en.wikipedia.org/wiki/Bluetooth_Low_Energy), two approaches are therefore possible:
encapsulating CoAP over BLE, or bringing up an IP stack on top of BLE.

There is no standard to encapsulate CoAP over BLE [GATT](https://en.wikipedia.org/wiki/Bluetooth_Low_Energy#Software_model), the client/server part of the BLE protocol.
The following solutions have been proposed:

- a [Samsung research paper](https://ieeexplore.ieee.org/abstract/document/8190936) (2017, no code),
- a [draft proposal](https://datatracker.ietf.org/doc/html/draft-amsuess-core-coap-over-gatt) in [IETF CoRE WG](https://datatracker.ietf.org/wg/core/about/)  (2020, [slides](https://datatracker.ietf.org/doc/slides-109-core-sessb-coap-over-gatt/), [code](https://gitlab.com/chrysn/coap-gatt-demo)),
- an implementation using the [standard layers SLIP and UART for encoding](https://www.maibornwolff.de/en/blog/talk-coap-me-iot-over-bluetooth-low-energy) (2021, some code snippets).

The CoRE draft sounds promising, but the author was waiting to see if browser vendors agreed on a [W3C specification for BLE](https://developer.mozilla.org/en-US/docs/Web/API/Web_Bluetooth_API) —
and Mozilla is [not too keen](https://github.com/mozilla/standards-positions/issues/95) on accepting Chrome's proposal.
A reimplemantation using the SLIP+UART approach would probably be reasonably straightforward, but not using out-of-the-box standards.

On the other hand, there is a well-known standard for IPv6 over BLE: [6LoWPAN](https://en.wikipedia.org/wiki/6LoWPAN) over BLE ([RFC 7668](https://datatracker.ietf.org/doc/html/rfc7668)).
The computer side could be implemented with [bluetooth_6lowpand](https://github.com/NordicSemiconductor/Linux-ble-6lowpan-joiner) and the Linux `bluetooth_6lowpan` kernel module.
There is currently no implementation for Android, but this may change soon:
[Alexander Wieland](https://diglib.tugraz.at/download.php?id=60a4ea6a34af4&location=browse) wrote one in 2020 as part of his master thesis, and seems willing to open-source it in the near future.
His implementation used introspection to access a private BLE API of Android (`CreateL2capChannel`), but Android 10 made the API official, allowing future-proof use.

{{< figure src="TPM-BLE.png" title="TPM remote attestation over BLE stack" >}}

## Existing solutions

There is a number of tools to perform remote attestation over IP.
For instance, [go-tpm-tools](https://github.com/google/go-tpm-tools) provide a self-contained solution in Go for both the client and server.
[go-attestation](https://github.com/google/go-attestation/) builds upon it to manage the certificates attesting a device identity.
However, they are not directly useful for local validation.

[Matthew Garrett](https://mjg59.dreamwidth.org/), who worked in the teams building those tools during his tenure at Google,
prototyped several techniques to perform remote attestation locally.
He introduced the idea of [TPM TOTP](https://mjg59.dreamwidth.org/35742.html), using a Timed One-Time-Password to confirm the state of the TPM
It was later re-implemented as a stand-alone tool, [tpm2-totp](https://github.com/tpm2-software/tpm2-totp),
and [is used by Trammell Hudson](https://trmm.net/Tpmtotp/) for his secure open-source firmware project [Heads](https://osresearch.net/).
Trammell's other project, [safeboot](https://safeboot.dev/) also provides tools for [remote attestation](https://safeboot.dev/attestation/) but they are not used for verification during the boot phase as far as I can tell.

Matthew Garrett also implemented [a prototype of remote attestation over BLE](https://mjg59.dreamwidth.org/54203.html), which was the initial inspiration for this blog post:
his [talk and demo](https://www.youtube.com/watch?v=FobfM9S9xSI) give good insights into why this approach is preferable to TPM-TOTP (in particular it provides more flexible policies, and a better understanding of what changed in case the laptop is compromised).

[Fobnail](https://fobnail.3mdeb.com/) is a research project by [3mdeb](https://3mdeb.com/) to implement a USB token performing remote attestation.
They want to use [CHARRA](https://github.com/Fraunhofer-SIT/charra) for the remote attestation, but the project is still in a very early stage.
Their milestones do not explain precisely how they plan to encapsulate CHARRA (CoAP) over USB.[^fobnail]
You can watch [their talk at the latest Trenchboot summit](https://www.youtube.com/watch?v=xZoCtNV8Qs0&t=3660s),
with a discussion at the end about replacing CHARRA with a minimalistic protocol over NFC, USB or BLE.

[^fobnail]: Update: the developpers told me that they assume IP communication and expect to use a USB network device on top of which they will run an IP stack in Fobnail.

## Next steps

I think the CoAP protocol defined in [draft-ietf-rats-reference-interaction-model](https://datatracker.ietf.org/doc/html/draft-ietf-rats-reference-interaction-models#appendix-A) is a good stepping stone for this work.
As explained above, it could be implemented either by encapsulating CoAP directly over BLE, or by running a 6LoWPAN IPv6 stack.
It is not entirely clear which approach is preferable here, and it would be worthwhile to experiment with both.
It may be less hassle and lower overhead to implement a self-contained solution using encapsulation than trying to bring up a 6LoWPAN daemon in initramfs,
but relying on existing standards has the benefit of exercising them in the field, and making them easier to re-use for the next application we come up with.

Once we get a basic prototype working, a lot of thought needs to be given to the provisioning and attestation UI.
Event logs are complex beasts, and it's not obvious to me what PCRs and events you want to validate.
The [reference integrity manifests](https://datatracker.ietf.org/doc/html/draft-ietf-rats-tpm-based-network-device-attest#section-2.4.1)
defined by RATS may prove useful for this step, but I haven't studied them enough to have a clear opinion yet.

If you are interested in the topic, please reach out to me. If I got anything wrong, let me know!
I am hoping to host an intern working on this topic at some point in the future, so if you are a student in a French instution, don't hesitate to get in touch.


