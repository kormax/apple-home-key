# Apple Home Key

<p float="left">
 <img src="./assets/HOME.KEY.PHONE.DEMO.webp" alt="![Reading a Home Key from an iPhone with a PN532]" width=250px>
 <img src="./assets/HOME.KEY.WATCH.DEMO.webp" alt="![Reading a Home Key from a Watch with a PN532]" width=250px>
 <img src="./assets/HOME.KEY.FAST.DEMO.webp" alt="![Home Key reading logs]" width=250px>
</p>


> [!NOTE]  
> [For fully functionall Apple Home Key Reader Implementation, visit this link](https://github.com/kormax/apple-home-key-reader)

# Overview


Apple Home Key is an NFC protocol used by select HomeKit certified locks to authenticate a user using a virtual key provisioned into their Apple device.  

There are three main components to the standard:
- Network:
  - HAP: Used to provision reader private keys, issuer keys, configure public keys of the devices that are given access to a particular Apple Home instance;
- NFC:
  - Authentication: runs on the SE and is inspired by the [Car Key](https://developer.apple.com/videos/play/wwdc2020/10006/) protocol, albeit with some changes, such as lack of direct pairing (which is done via HAP instead), changes to the KDF influcenced by multi-reader environment and other small ones;
  - Attestation exchange: implemented via HCE and uses ISO18013 engagement and NFC data transfer.  

<br>

Just like parent protocols, Home Key has following advantages:
- This protocol provides reader authentication, data encryption, forward secrecy;
- It [allows to share keys*](#key-sharing), although this functionality is not implemented;
- In theory, future locks could implement UWB-based access as it's supported by specification.


# Terminology

- Nonce - Single-use number;
- EC - Elliptic Curve;
- Endpoint - end device;
- Issuer - a party that's generating keys;
- If a key is mentioned without algorithm info, assume its `secp256r1`;
- Copernicus - codename of the Home Key applet implementation.


# HomeKit


For HAP part of the Home Key, refer to comprehensive overview done by [@kupa22](https://github.com/kupa22/apple-homekey).

<sub>Previously this section contained examples of some requests with possible meaning for each of their parts. With new knowledge, previous assumptions have been deemed to be misleading, and thus have been removed.</sub>


## Key Color

**HardwareFinish of the first lock added to your home installation influences the color of the lock art for the whole home.**

Following finish variations are mentioned in IOS system files:

   | Color  | Art                                                                                  | Value      | Notes                                                                                                |
   | ------ | ------------------------------------------------------------------------------------ | ---------- | ---------------------------------------------------------------------------------------------------- |
   | Black  | <img src="./assets/HOME.KEY.COLOR.BLACK.webp" alt="![Black Home Key]" width=200px>   | `00 00 00` |                                                                                                      |
   | Gold   | <img src="./assets/HOME.KEY.COLOR.GOLD.webp" alt="![Gold Home Key]" width=200px>     | `AA D6 EC` |                                                                                                      |
   | Silver | <img src="./assets/HOME.KEY.COLOR.SILVER.webp" alt="![Silver Home Key]" width=200px> | `E3 E3 E3` |                                                                                                      |
   | Tan    | <img src="./assets/HOME.KEY.COLOR.TAN.webp" alt="![Tan Home Key]" width=200px>       | `CE D5 DA` | The default color. If an unexisting color combo is chosen, this color will be selected as a fallback |

<sub>`00` also has to be appended to color value</sub>  
<sub>Credit to [@kupa22](https://github.com/kupa22/apple-homekey) for finding rest of the key colors</sub>

# NFC

## ECP

[ECP](https://github.com/kormax/apple-enhanced-contactless-polling) allows Home Keys to work via express mode and is required to be implemented by certified locks.

Full Home Key ECP frame looks like this

```
   6a 02 cb 02 06 021100 deadbeefdeadbeef
```

Following characteristics can be noted:
- It belongs to the Access(`02`) reader group  with a dedicated subtype HomeKey(`06`);
- It contains a single 3-byte TCI with a value of `02 11 00`, no other variations have been met. IOS file system contains multiple copies of a Home Key pass json with this TCI;
- The final 8 bytes of an ECP Home Key frame contain reader group identifier, which allows IOS to differentiate between keys for different Home installations.  

(NOTE) Actual reader identifier is 16 bytes long, first 8 bytes are the same for all locks in a single home, while the latter 8 are unique to each one. First 8 are used for ECP.  
(BUG) If more than one Home Key is added to a device, ECP stops working correctly, as a device responds to any reader emitting Home Key ECP frame even if the reader identifier is not known to it.  
(BUG) If you disable express mode for a single key while having multiple with enabled express mode, it won't appear on a screen when brought near to a reader. Disabling express mode for all home keys fixes this issue.

## Applets and Application Identifiers

Home Key uses two application identifiers:
1. Home Key Primary:  
    `A0000008580101`. Used for authentication. Implemented on the SE;
2. Home Key Configuration:  
    `A0000008580102`. Used for attestation exchange. Implemented via HCE. Can only be selected after a successful STANDARD authentication with a following EXCHANGE and CONTROL FLOW commands.

In most situations a reader will only use the Primary applet.  
Configuration applet will be selected only if a new device has been invited and a key data hasn't been provisioned into a lock prior to that;
Invitation is attested by the HAP pairing key of the person whom the device belongs to.


## Command overview

   | Command name                | CLA  | INS  | P1    | P2   | DATA                                                 | LE   | Response Data                                        | Notes                                                                                            |
   | --------------------------- | ---- | ---- | ----- | ---- | ---------------------------------------------------- | ---- | ---------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
   | SELECT USER APPLET          | `00` | `A4` | `04`  | `00` | Home Key AID                                         | `00` | BER-TLV encoded data                                 |                                                                                                  |
   | FAST (AUTH0)                | `80` | `80` | FLAGS | TYPE | BER-TLV encoded data                                 | `00` | BER-TLV encoded data                                 | Data format described below                                                                      |
   | STANDARD (AUTH1)            | `80` | `81` | `00`  | `00` | BER-TLV encoded data                                 |      | Encrypted BER-TLV encoded data                       | Data format described below                                                                      |
   | EXCHANGE                    | `84` | `c9` | `00`  | `00` | Encrypted BER-TLV encoded data                       |      | Optional Encrypted BER-TLV encoded data              | Data format described below                                                                      |
   | CONTROL FLOW                | `80` | `3c` | STEP  | INFO | None                                                 |      | None                                                 | No data, used purely for UX                                                                      |
   | SELECT CONFIGURATION APPLET | `00` | `a4` | `04`  | `00` | Home Key Configuration AID                           | `00` | None                                                 |                                                                                                  |
   | ENVELOPE                    | `00` | `c3` | `00`  | FRMT | BER-TLV with nested NDEF or CBOR with encrypted data | `00` | BER-TLV with nested NDEF or CBOR with encrypted data | Data format described below                                                                      |
   | GET RESPONSE                | `00` | `c0` | `00`  | `00` | None                                                 | `XX` | CBOR with encrypted data                             | LE is length of data expected in response. Response data format as the one requested in ENVELOPE |


Commands are executed in a following sequence:
1. SELECT USER APPLET:  
    Reader transmits Home Key AID; Device responds with a version list TLV;
    Reader has to verify that there is a protocol version match between a list provided by a device and itself;
2. FAST:  
    Reader transmits required protocol version, transaction nonce, ephemeral public `secp256r1` key, and its identifier;  
    Device responds with an authentication cryptogram and its own ephemeral public `secp256r1` key;  
    Reader tries to verify that cryptogram was generated by a known device with a beforementioned encryption key;  
    A) If cryptograms match and there is no data to synchronize, authentication is confirmed and session is completed;  
    B) If there's a cryptogram mismatch or there is data to synchronize, we continue the command flow;
3. STANDARD:  
    Reader combines keys, nonces, other data exchanged during the transaction and signs it using its own private key;
    A device verifies that a signature is valid, and returns an encrypted payload containing signature of the transaction data using the device key;
    Common keys are established during this step to be used in FAST command in next communications. 
4. EXCHANGE:  
    Using the established encrypted channel, reader can request a switch to a configuration applet, by providing a shared secret to be used during that operation.  
    In cases when a reader does not recognize the device, the communication continues further with configuration.
*  CONTROL FLOW:
    Used in between other commands to communicate transaction state to the device. This command is reponsible for UX, such as:
    - Success checkmark;
    - Failure exclamation mark;
    - Error messages;
    - Notify about a switch to configuration applet;
5. SELECT CONFIGURATION APPLET:  
    Reader transmits Home Key Configuration AID. Device responds positively without any data;
    Re-selection is done because the device has to switch routing from the SE to HCE for attestation transfer.
6. ENVELOPE:  
    Reader transmits BER-TLV-encoded data with nested CBOR or NDEF messages.  
    First ENVELOPE command and response pair mimics ISO18013 NFC handover with NDEF.  
    Following command-response pairs mimic CBOR data transfer with nested encrypted data. Data format is the same as in ISO18013.  
    Useful payload part contains device's `secp256r1` public key used by Home Key, attested by the public `ed25519` HAP key of device owner. 
* GET RESPONSE:
    If a response to `ENVELOPE` or `GET RESPONSE` had an sw `6100`, reader uses this command to request additional response parts.
  
Alternative overview of the NFC transaction flow can be seen [here](https://github.com/kupa22/apple-homekey/#homekey-transaction-overview).

## Commands

### SELECT USER APPLET

#### Request

##### Overview

| CLA  | INS  | P1   | P2   | DATA             | LE   |
| ---- | ---- | ---- | ---- | ---------------- | ---- |
| `00` | `A4` | `04` | `00` | `A0000008580101` | `00` |

#### Response 

##### Overview

| DATA                 | SW1  | SW2  |
| -------------------- | ---- | ---- |
| Refer to data format | `90` | `00` |

Any response rather than `9000` means that applet is not available

##### Data format

| Name               | Tag  | Length | Example    | Notes                                        |
| ------------------ | ---- | ------ | ---------- | -------------------------------------------- |
| Supported versions | `5c` | 2*n    | `02000100` | First byte is major version, second is minor |

Currently only versions `0100`  and `0200` aka 1.0 and 2.0 are known

##### Data example
```
5c[04]: 
  02000100   # Supports version 2.0, and version 1.0
```

### FAST

#### Request

##### Overview

| CLA  | INS  | P1    | P2   | DATA                 | LE   |
| ---- | ---- | ----- | ---- | -------------------- | ---- |
| `80` | `80` | FLAGS | `00` | Refer to data format | None |

FLAG and TYPE parameters seem to correlate with overall transaction length;

Flag:
- `00` if intending to use STANDARD command (no cryptogram will be generated in response);
- `01` if trying to using FAST command only (still can continue with STANDARD if fail).

##### Data format

| Name                        | Tag  | Length | Example                            | Notes                                                                           |
| --------------------------- | ---- | ------ | ---------------------------------- | ------------------------------------------------------------------------------- |
| Selected protocol version   | `5c` | 2      | `0200`                             | First byte is major version, second is minor                                    |
| Reader ephemeral public key | `87` | 65     | `04`...                            | Contains an uncompressed `secp256r1` public key                                 |
| Transaction nonce           | `4c` | 16     | `deadbeefdeadbeefdeadbeefdeadbeef` | A random number used to verify that responses are generated during this session |
| Reader identifier           | `4d` | 16     | `deadbeefdeadbeefdeadbeefdeadbeef` | First 8 bytes is reader group, last 8 are unique to the reader                  |

#### Response

##### Overview

| DATA                 | SW1  | SW2  |
| -------------------- | ---- | ---- |
| Refer to data format | `90` | `00` |

Status other than `9000` cannot be encountered 

##### Data format

| Name                        | Tag  | Length | Example                            | Notes                                  |
| --------------------------- | ---- | ------ | ---------------------------------- | -------------------------------------- |
| Device ephemeral public key | `86` | 65     | `04`...                            | Uncompressed `secp256r1` public key    |
| Authentication cryptogram   | `9d` | 16     | `7656a6256aee6f9bdc55ed45d96026a3` | Optional, returned only if p1 is  `01` |


##### Data example
```
86[41]: 
  046e197441b017a6452dfe33a3645860c09a7fb34f3e84c9d6a834c737fe4e4185b37cccc2004b9cb08f837b0920d42c59ab1ce403a95cefdfe221120175f82218
9d[10]:
  7656a6256aee6f9bdc55ed45d96026a3
```


### STANDARD

> [!NOTE]
> Commands from this point on forward are not fully documented, [@kupa22's research](https://github.com/kupa22/apple-homekey?tab=readme-ov-file#homekey-transaction-overview) contains some missing pieces and pointers.  
> This section will soon be completed, please bear with us.

#### Request

##### Overview

| CLA  | INS  | P1   | P2   | DATA                 | LE   |
| ---- | ---- | ---- | ---- | -------------------- | ---- |
| `80` | `81` | `00` | `00` | Refer to data format | None |


##### Data format
| Name                                     | Tag  | Length | Example | Notes |
| ---------------------------------------- | ---- | ------ | ------- | ----- |
| Signature over shared info in point form | `9e` | 64     |         |       |

##### Data example
```
9e[40]:
  57a071cfeeff242878c68ef02fc430fe59cbf56741a1cadfcb0b23f962723d7321b67ab65015d50688edd17e7e658f4f6547b79bcbf9024a3bf701c216256050
```

#### Response

##### Overview

| DATA                                                         | SW1  | SW2  |
| ------------------------------------------------------------ | ---- | ---- |
| Encrypted. Exact format unknown, although there some guesses | `90` | `00` |

### COMMAND FLOW

#### Request

##### Overview

| CLA  | INS  | P1     | P2   | DATA | LE   |
| ---- | ---- | ------ | ---- | ---- | ---- |
| `80` | `3c` | STATUS | STEP | None | None |

SUCCESS is a flag that indicates transaction status:
- `00` - Failure;
- `01` - Success (Checkmark will appear);
- `40` - Need to switch to HCE applet for attestation exchange.

STEP is a second flag:
- `00` - Failure or success in normal circumstances;
- `a0` - Need to switch to HCE applet for attestation exchange.

#### Response

##### Overview

| DATA | SW1  | SW2  |
| ---- | ---- | ---- |
| None | `90` | `00` |


### SELECT CONFIGURATION APPLET

#### Request

##### Overview

| CLA  | INS  | P1   | P2   | DATA             | LE   |
| ---- | ---- | ---- | ---- | ---------------- | ---- |
| `00` | `A4` | `04` | `00` | `a0000008580102` | `00` |

#### Response 

##### Overview

| DATA | SW1  | SW2  |
| ---- | ---- | ---- |
| None | `90` | `00` |

Any response rather than `9000` means that applet is not available.
This applet is only selectable after EXCHAGNE + CONTROL FLOW (need attestation) commands have been successfuly used.  
otherwise the device will present an error and stop NFC communication.


### ENVELOPE

#### Request

##### Overview

| CLA  | INS  | P1   | P2   | DATA                                          | LE   |
| ---- | ---- | ---- | ---- | --------------------------------------------- | ---- |
| `00` | `c3` | `00` | FRMT | BER-TLV-encoded data with nested NDEF or CBOR | `00` |

FRMT real meaning is unknown, but according to observations it means the following:
- `01` NDEF nfc handover message in command and response (first command-response pair only);
- `00` CBOR message in command in response (other command-response pairs). 

#### Response 

##### Overview

| DATA                                          | SW1  | SW2  |
| --------------------------------------------- | ---- | ---- |
| BER-TLV-encoded data with nested NDEF or CBOR | `XX` | `XX` |

Status words:
   | SW1  | SW2  | Notes                                                                                  |
   | ---- | ---- | -------------------------------------------------------------------------------------- |
   | `90` | `00` | Data returned in full                                                                  |
   | `61` | `XX` | More data can be requested (00 also valid value meaning that more than 255 bytes left) |

### GET RESPONSE

#### Request

##### Overview

| CLA  | INS  | P1   | P2   | DATA | LE   |
| ---- | ---- | ---- | ---- | ---- | ---- |
| `00` | `c0` | `00` | `00` | None | `XX` |

LE should be equal to the value returned in the previous response to `ENVELOPE` or `GET RESPONSE`;

#### Response 

##### Overview

| DATA                  | SW1  | SW2  |
| --------------------- | ---- | ---- |
| TLV-encoded data/part | `XX` | `XX` |

Status words:
   | SW1  | SW2  | Notes                                                                                         |
   | ---- | ---- | --------------------------------------------------------------------------------------------- |
   | `90` | `00` | Data returned in full                                                                         |
   | `61` | `XX` | More data can be requested (00 also valid value meaning that more than 255 bytes left)                                              |


## Communication examples

Communication examples can be viewed in [resources directory](./resources/README.md).


## Extras

### Key Sharing

Unlike with Car Keys, where invited keys are attested by the key of a car owner, each Home Key is attested by the HAP (AKA issuer) keys of each particular person.  
This has following advantages:
  - People may be removed from Apple Home at will, as it woudn't break the attestation chain for everyone else (only the HAP-issuer key of removed person will be removed);
  - It allows for a lock to support much more enrolled devices, as the lock really only needs to permamentely store issuer keys, and a single one can attest multiple devices at a time (phone + watch).  
     Although in reality the lock still stores some amount of credentials locally (current locks report up to 16), as attestation exchange takes much longer than FAST auth (0.2sec vs 1sec);   

But there are also disadvantages:
  - Home Key protocol, unlike Car Key protocol, does not support "free-for-all" sharing natively, as it assumes that each key is bound to a particular HAP key, which automatically requires someone to be present in your Apple Home.  
  
For people who still expect sharing to happen, not all hope is lost, as:
  - Standard seems to have provisions for adding "Issuer keys" into a lock independently of HAP, but it is not known if it is actually implemented by the existing locks. 
  - Key sharing may be implemented via stub/fake/invisible HomeKit users, where an issuer key of a fake user will be used to attest a key of an invited person;

Only the time will tell if they are going to do any of that;

### Limits

During the tests, it has been found out that a single device can store up to 16 persistent keys used for FAST authentication.  
The keys are overwriten in a FIFO/Cyclic fashion, with the oldest unused one removed the first.
Even with this limitation, the possible amount of lock inside a single household is virtually unlimited, as your device will still fallback to a STANDARD authentication or even an attestation exchange if needed.

### Configuration applet

Configuration applet, utilized for attestation package retrieval, is belived to be implemented via HCE. The reasons for that are the following:
1. It doesn't work in power reserve mode, with NFC communication hanging on reselection;
2. The same attestation package transferred during communication can be found in the .pkpass file of the key inside of the normal file system;  

That would explain the need for additional applet reselection, as this is one of the ways of returning control to the OS in order to give it an ability to read the package file and send it via NFC.   
Also, storing a static 0.6-1kb package inside of the highly limited (1mb?) SE memory would've been wasteful.

It raises a question if the so simillar Digital Car Key dual-applet system works the same way.


### Copernicus

> Nicolaus Copernicus was a Renaissance polymath, active as a mathematician, astronomer, and Catholic canon.  - Wikipedia.

Besides, Copernicus is the codename of the applet used by Apple Home Keys.  

Apple OS internals contain not one, but three codenames that start with such a name:
- `CopernicusCar`;
- `CopernicusHome`;
- `CopernicusAccess`.

Taking a deeper look, we find out that all of those applets share the same primary internal AID `A000000704D011500000000002000000` and container AID `A000000704D011500000000001000000`, which also **means that they share the same implementation**. 

This explains why there's so much overlap between the Car Key protocol (as described in an overview video), and a Home Key protocol. Because it's the same code base, just with some features/quirks for each subtype turned on and off for particular instances.   

It also tells us that there's a third, publically unknown "Access" implementation.  
Not one **based not on Seos (offices) or Mifare (hotels, offices, theme parks)**, but on the same "Key" standard.  
This "Access" applet could be the one used in solutions provided to Hotels and managed Properties, the ones which have built-in sharing functionality. The only one real-life implementation of this protocol variation was presumably done by [Salto](https://support.saltosystems.com/homelok/user-guide/keyholder/faqs/apple-wallet-keys/), as they are the only ones who offer native in-wallet key sharing.

It also explains why the attestation exchange part of the Home Key protocol seemed too complex for the task it was aiming to achieve, especially the storage of validity dates, floor and room access flags, etc.  
In the context of shared/managed properties all of that info could be used for "offline synchronization", as an endpoint would be able to prove access to a particular zone of the property without a need for a reader to ring a backend and continuously synchronize a list of allowed users. 


### Unified Access

UnifiedAccess is the Passkit framework codename for Copernicus-based access passes.


### Purple Trust

Both the Home and Access Keys have system references which start with `purpleTrustAir`.  
Considering that Car Keys have no such references, my guess is that it could be related to networked pairing or Bluetooth subsystems (air), which are unused in the former case.


### PTA

Car, Home, Access applets have references to them that start or end with `pta`.   

Coincidentally, Apple Cash Card is named `ptc`, which could mean that `pt` is a prefix for internally-developed applets, with `c` standing for `Cash` and `a` standing for `Access`.


### Role of ISO18013

Passkit framework defines following ISO18013 credential storage partitions:
- Identity
- UnifiedAccess

It proves that usage of ISO18013 by UnifiedAccess was not a "one-time quick hack" but more like "temporary solutions are permament".

Considering that ISO18013 for a Home Key can be found on a file system, it raises a question if it's the same for Identity, or if the different `Partition` part actually means that Identity document in stored somewhere else?


> [!NOTE]  
>  SEP (Secure Enclave Processor) and SE (Secure Element) are two different things. SE is a separate certified chip, usually packaged together with an NFC controller, which runs primitive but hardened JCOP OS implementation. SEP (TrustZone) is a part of the CPU, running a separate manufacturer-provided OS. While SEP boasts greater power compared to SE, it also presents a higher vulnerability risk. 


Indeed, Apple doesn't claim that ISO18013 Identity runs on the secure element. Instead, [they say](https://support.apple.com/en-gb/guide/security/secc50cff810/web):
> By storing the private key for ID authentication in the iPhone device’s Secure Element, the ID is bound to the same device that the state-issuing authority created the ID for"

> The information reflected on the user’s ID in Apple Wallet is stored in an encrypted format protected by the Secure Enclave.
 

Which means that base ISO18013 identity data is not stored in the secure element, but is instead stored in the file system either in an SEP-key-backed encrypted form, or a plain form, with SE only being used for crypto proof generation.  
So Apple's ISO18013 implementation runs exclusively via HCE, with SE serving only for key storage, so quite similar to newer Android devices with StrongBox? Eh :))?. 


### Aliro

Considering that the CCC Digital Car Key implementation is based primarily on work done by Apple, it's quite probable that the upcoming [Aliro](https://csa-iot.org/all-solutions/aliro/) standard will also borrow big chunks of current Apple Car/Home/Access key implementation.

The feature that could undego the most changes will be the "attestation exchange" part.  
Aliro is planned as "Multi-Architecture" and "Multi-purpose", so it will contain the same features needed by shared and private properties, such as:
- Sharing and offline-enabled invitations (Attestations):
  - Simplified non-ISO18013 based flow;
  - Access zone masks;
  - Date and validity periods;
  - Limitations on buddy/multiple devices, ability to share.
- Configuration synchronization: (Mailboxes):
  - Revocations;
  - Setting updates.
- Multiple radio standard support:
  - Bluetooth;
  - UWB.
 

### Applet size

All Copernicus-based Car/Home/Access applets share the same implementation with same memory usage regardless of instance type.  
According to Passkit framework (thanks to SE memory management screen), the usage is as follows (numbers in bytes):

| Type         | CLEAR_ON_DESELECT (cod) | CLEAR_ON_RESET (cor) | PERSISTANT_HEAP (pHeap) |
| ------------ | ----------------------- | -------------------- | ----------------------- |
| Selectable   | 124                     | 19                   | 7348                    |
| Package      | 0                       | 0                    | 54356                   |
| Personalized | 124                     | 0                    | 5900                    |
| Container    | 4203                    | 332                  | 1496                    |

Copernicus executable code (Package) takes 54356 bytes (54 KB), with each instance (Personalized) taking an additional 5900 bytes each.

Modern IOS devices have about 0.7MB of memory available for users, which means that a user will be able to add about `(700000 - 54356) / 5900 = 109` keys until the memory runs out.  
In reality, due to nature of JCOP GC and other things (such as additional provisioning for Selectable and Container memory, which I'm not entirely sure about, tell me if you know), the real number could differ.  
Also, considering that a real user will install other applets, this amount could drop significantly, but regardless of that no normal person will ever reach the limit.


# Notes

- If you find any mistakes/typos or have extra information to add, feel free to raise an issue or create a PR;
- Please be aware that the information presented here is based on personal findings, observations and assumptions, so there's no guarantee that it's correct in its entirety.

# References

* Resources that helped with research:
  - Parallel Home Key protocol research project:
    - [Apple HomeKey - @kupa22](https://github.com/kupa22/apple-homekey);
  - Original inspiration:
    - [KhaosT - Home Key demo](https://twitter.com/followhomekit/status/1478489402028531712) [(Archive)](https://web.archive.org);
  - General:
    - [TLV8 format](https://pypi.org/project/tlv8/);
    - [List of digital keys in mobile wallets](https://en.wikipedia.org/wiki/List_of_digital_keys_in_mobile_wallets);
    - [IPSW.me](https://ipsw.me/product/iPhone) - used to get IOS IPSW to look for clues in a file system;
    - [TLV Utilities](https://emvlab.org/tlvutils/);
    - [ASN1 decoder](https://lapo.it/asn1js/);
    - [NDEF parser](https://ndefparser.online);
    - [CBOR playground](https://cbor.nemo157.com).
  - Source code:  
    - [Google: Identity credential](https://github.com/google/identity-credential).
  - Apple resources:
    - [Apple Developer Documentation](https://developer.apple.com/documentation/);
    - [WWDC Introducting "Home Keys" (I'm not kidding)](https://developer.apple.com/videos/play/wwdc2020/10006/);
    - [Unlock your door with a home key on iPhone](https://support.apple.com/en-gb/guide/iphone/iph0dc255875/ios);
  - Device brochures:  
    - [Aqara A100 Zigbee](https://www.aqara.com/en/product/smart-door-lock-a100-zigbee);
    - [Aqara D100 Zigbee](https://www.aqara.com/en/product/smart-door-lock-d100-zigbee);
    - [Aqara Smart Lock U100](https://www.aqara.com/en/product/smart-lock-u100);
    - [Schlage encode plus](https://www.schlage.com/en/home/smart-locks/encode-plus.html);
    - [Level Lock +](https://level.co/shop/level-lock-plus);
    - [TUO Smart Lock](https://findtuo.com/pages/smart-lock).
  - Other:
    - [Aliro](https://csa-iot.org/all-solutions/aliro/);
    - [Salto Homelok - Apple Wallet Keys](https://support.saltosystems.com/homelok/user-guide/keyholder/faqs/apple-wallet-keys/);
* Devices and software used for analysis:
  - [HAP-NodeJS](https://github.com/homebridge/HAP-NodeJS) - used to create virtual lock instances in order to look at communication and data structures;
  - Proxmark3 Easy - used to sniff Home Key transactions. Proxmark3 RDV2/4 can also be used;
  - [Proxmark3 Iceman Fork](https://github.com/RfidResearchGroup/proxmark3) - firmware for Proxmark3.
