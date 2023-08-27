# Google Smart Tap


<p float="left">
<img src="./assets/SMARTTAP.VASONLY.DEMO.webp" alt="![Smart Tap VAS only]" width=250px>
 <img src="./assets/SMARTTAP.VASANDPAY.DEMO.webp" alt="![Smart Tap VAS and payment]" width=250px>
</p>


# Overview


Google Smart Tap is a proprietary NFC protocol that can be used for sending data from a mobile device to an NFC terminal.

Data is conveyed from the device to the terminal in encrypted form, using keys derived during the channel negotiation phase. At the moment of negotiation, the reader sends a device its collector id, key version, and a signature of derived data signed using the collector key, thus proving to the device that the reader is allowed to get the information.

Only one pass (object) could be conveyed during a single tap (single read).  
If more than one pass is eligible for redemption, a selection carousel will appear and a user will be prompted to tap again.

Version 2.1 was current at the time of writing.


# Application identifiers


Smart Tap can be activated using multiple application ids (AID):

1. Universal VAS AID (hex of encoded `OSE.VAS.01`), also used by Apple VAS.  
    ```
    4f53452e5641532e3031
    ```
2. Smart Tap 1 (Deprecated, does not work anymore)
    ```
    a000000476d0000101
    ```
3. Smart Tap 2
    ```
    a000000476d0000111
    ```

The ususal implementation for most readers is to select `OSE.VAS.01` in order to detect what wallet provider is available on device (stored in TLV tag 50), if "AndroidPay" is the value, then we have a device with Google Wallet, and Smart Tap 2 can be reselected if required.  
As of version 2.1 device nonce and key is returned in OSE, so a separate selection of Smart Tap is not needed.


# Command overview

SmartTap-exclusive commands and responses use multi-layer nested NDEF messages and records for conveying information. 
As of version 2.1 following commands are available:

| Command name             | CLA | INS | P1  | P2  | DATA                                  | LE  | Response data                                                     | NOTES                                                                                                                                                                                    |
| ------------------------ | --- | --- | --- | --- | ------------------------------------- | --- | ----------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| SELECT VAS APPLET        | 00  | A4  | 04  | 00  | VAS AID                               | 00  | BER-TLV                                                           | Optional. Should be implemented if you want to support other wallets with value-added services, as this AID allows to find out what implementation is used without bruteforce            |
| SELECT SMART TAP APPLET  | 00  | A4  | 04  | 00  | Smart Tap VX AID                      | 00  | NDEF message                                                      | Optional if SELECT VAS APPLET has been used.                                                                                                                                             |
| NEGOTIATE SECURE CHANNEL | 90  | 53  | 00  | 00  | NDEF message with nested NDEF message | 00  | NDEF message with nested NDEF                                     |                                                                                                                                                                                          |
| GET DATA                 | 90  | 50  | 00  | 00  | NDEF message with nested NDEF message | 00  | Part of NDEF message with encrypted and/or compressed nested NDEF | Can be used only after channel negotiation                                                                                                                                               |
| GET MORE DATA            | 90  | C0  | 00  | 00  | No data (V 2.1)                       | 00  | Part of NDEF message with encrypted and/or compressed nested NDEF | Can be used if GET DATA response sw is 9100                                                                                                                                              |
| PUSH DATA                | 90  | 52  | 00  | 00  | NDEF message with nested NDEF         | 00  | NDEF message with nested NDEF                                     | Can be used before of after data read. Secure channel not needed. Exact use case of this command is not known. Possible use is to push signup URL but this feature seems to be disabled. |

Commands are executed as follows:
1. SELECT VAS APPLET:  
   Optional. May be the first command in a read flow.
   Reader transmits universal VAS AID; Device response with wallet implementation name in TLV tag `50`.  
   If value is `416e64726f6964506179`, which is a value of `AndroidPay` string in ASCII-encoded form, we have a device that supports SmartTap.  
   Device returns supported versions, other info. For, newer smart tap implementations it returns device nonce and device ephemeral key.
2. SELECT SMART TAP APPLET:  
   Optional. May be the first command in a read flow, or a fallback if SELECT VAS APPLET returned no device nonce or device ephemeral key;
   Device returns version support range, and a device nonce.
3. NEGOTIATE SECURE CHANNEL:  
   Reader generates a nonce, ephemeral public key. Reader generates a signature over concatenation of reader nonce, device nonce, collector id, and reader ephemeral public ke using a private key of collector.  
   It then transmits session-related information together with the signature to the end device, proving to it that the reader is owned by a particular pass collector.
   Device responds with ephemeral public key, session information.
   Reader then uses all collected information in order to calculate session keys that will be used to encrypt and decrypt data.
4. GET DATA:  
   Reader transmits its configuration, capabilityies, session information.
   Device responds with full or partial NDEF data with encrypted nested data.
5. GET MORE DATA:  
   If a response to previous GET DATA or GET MORE DATA returned status word `91 00`, then more data has to be read.
   Reader transmits this command as many times as needed, until a device responds any response rather than `91 00`.
6. PUSH DATA:  
   This command is not known to be used by any IRL readers. It was intended to be used by readers in order to push some information, like sign-up urls, pass data updates, pass addition, etc.  
   None of the possible parameter combinations seem to have any visible effects, so theres a big chance that its a leftover of unfinished or cut functionality.
   Reader transmits session info, list of service statuses (changes to the passes, usage history, URL signups), total transaction info (ability to send receipts).
   Device responds with an acknowledgment and session info.



# Entities, constants and data format

SmartTap protocol uses NDEF records for transmitting data, sometimes in nested fashion.  
Before closer look at communication, it is important to undersand data representation used during communication.


## Data format

Some records may contain a data format identifier in the first byte of payload

Following data formats are known:

| Name        | Field |
| ----------- | ----- |
| UNSPECIFIED | 0x00  |
| ASCII       | 0x01  |
| UTF_8       | 0x02  |
| UTF_16      | 0x03  |
| BINARY      | 0x04  |
| BCD         | 0x05  |

Most common used type is BINARY, but even it is not used for all payloads.  
There seems to be no particular pattern as for where data format identifier is mandatory, so its use will be mentioned for each record type.

## Record types

Type in SmartTap defines what kind of object/data does the record hold. Each type is denoted by a specific short string.
Depending on TNF, type value may be populated into either type or id fields:
Type field follows following rules when being populated into NDEF Records

| TNF              | Field |
| ---------------- | ----- |
| WELL_KNOWN(0x01) | id    |
| EXTERNAL(0x04)   | type  |

Older SmartTap versions required `id` field to be used instead to populate type, but new ones recognize only the described rules.


## Records 

Following types exist in SmartTap, but not all of them are used:

| Name                        | Type | Payload                                                                                                                                                        |
| --------------------------- | ---- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| HANDSET_NONCE               | mdn  | BINARY format flag + 32 byte long nonce                                                                                                                        |
| SESSION                     | ses  | 8 byte long id, 1 byte long sequence counter, 1 byte long status                                                                                               |
| NEGOTIATE_REQUEST           | ngr  | 2 byte long version, nested NDEF message with `ses` and `cpr` records                                                                                          |
| NEGOTIATE_RESPONSE          | nrs  | Nested NDEF message with `ses` and `dpk` records                                                                                                               |
| CRYPTO_PARAMS               | cpr  | 32 byte long reader nonce, 1 byte long auth flag, 33 byte long reader ephemeral public key, 4 byte long key version, NDEF message with `sig` and `cld` records |
| SIGNATURE                   | sig  | BINARY format flag + 72 byte long signature data encoded in ASN1 as Dss-Sig-Value                                                                              |
| SERVICE_REQUEST             | srq  | 2 byte long version, nested NDEF message with `ses`, `mer`, `slr`, `pcr` records                                                                               |
| ADDITIONAL_SERVICE          | asr  |                                                                                                                                                                |
| SERVICE_RESPONSE            | srs  | Contains nested `ses` and `reb` records                                                                                                                        |
| MERCHANT                    | mer  | Nested NDEF message with mandatory `cld` and optional `lid`, `tid`, `mnr`, `mcr` records                                                                       |
| COLLECTOR_ID_V0             | mid  |                                                                                                                                                                |
| COLLECTOR_ID                | cld  | 4 byte long big endian representation of collector id number                                                                                                   |
| LOCATION_ID                 | lid  |                                                                                                                                                                |
| TERMINAL_ID                 | tid  |                                                                                                                                                                |
| MERCHANT_NAME               | mnr  |                                                                                                                                                                |
| MERCHANT_CATEGORY           | mcr  |                                                                                                                                                                |
| SERVICE_LIST                | slr  | Nested NDEF message with `str` record                                                                                                                          |
| SERVICE_TYPE_REQUEST        | str  | List of 1 byte long service objects types to request                                                                                                           |
| HANDSET_EPHERMAL_PUBLIC_KEY | dpk  | 33-byte long public ephemeral EC key                                                                                                                           |
| ENCRYPTED_SERVICE_VALUE     | enc  |                                                                                                                                                                |
| SERVICE_VALUE               | asv  | Contains `i` record. Depending on object type, may contain one of `cus`, `et`, `fl`, `gc`, `ly`, `of`, `pl`, `tr`, `gr` records.                               |
| SERVICE_ID                  | sid  |                                                                                                                                                                |
| OBJECT_ID                   | oid  | BINARY format flag + 8 byte identifier                                                                                                                         |
| RECORD_BUNDLE               | reb  | First byte is response flag, other bytes is encrypted and/or compressed data depending on flag, containing `asv` object records                                |
| CUSTOMER                    | cus  | May be contained in unpacked record bundle. Contains                                                                                                           |
| CUSTOMER_ID                 | cid  | BINARY format flag + 16 byte identifier                                                                                                                        |
| CUSTOMER_LANGUAGE           | cpl  | NDEF-encoded string with language code                                                                                                                         |
| CUSTOMER_TAP_ID             | cut  | BINARY format flag + 16 byte identifier                                                                                                                        |
| EVENT                       | et   | Object                                                                                                                                                         |
| FLIGHT                      | fl   | Object                                                                                                                                                         |
| GIFT_CARD                   | gc   | Object                                                                                                                                                         |
| LOYALTY                     | ly   | Object                                                                                                                                                         |
| OFFER                       | of   | Object                                                                                                                                                         |
| PLC                         | pl   | Object                                                                                                                                                         |
| TRANSIT                     | tr   | Object                                                                                                                                                         |
| GENERIC                     | gr   | Object                                                                                                                                                         |
| ISSUER                      | i    | BINARY format flag + 5 byte identifier                                                                                                                         |
| SERVICE_NUMBER              | n    | UNSPECIFIED format flag + variable length value of a particular service. In essence, this is the pass data.                                                    |
| TRANSACTION_COUNTER         | tcr  |                                                                                                                                                                |
| PIN                         | p    |                                                                                                                                                                |
| EXPIRATION_DATE             | ex   |                                                                                                                                                                |
| CVC                         | c1   |                                                                                                                                                                |
| POS_CAPABILITIES            | pcr  | Contains 5 byte long flag POS CAPABILITIES mask                                                                                                                |
| PUSH_SERVICE_REQUEST        | spr  | Nested NDEF message with mandatory `ses` and optional, `bpr`, `nsr`, `ssr` records                                                                             |
| PUSH_SERVICE_RESPONSE       | psr  | Nested NDEF message with `ses` record                                                                                                                          |
| SERVICE_STATUS              | ssr  | Nested NDEF message with `oid`, `sug` and `sup` records                                                                                                        |
| SERVICE_USAGE               | sug  | Nested NDEF message with `sut` and `sud` records                                                                                                               |
| SERVICE_USAGE_TITLE         | sut  | NDEF encoded text                                                                                                                                              |
| SERVICE_USAGE_DESCRIPTION   | sud  | NDEF encoded text                                                                                                                                              |
| SERVICE_UPDATE              | sup  | 1 byte long service operation code concatenated with value                                                                                                     |
| NEW_SERVICE                 | nsr  | Nested NDEF message with `nst` and `nsu` records                                                                                                               |
| NEW_SERVICE_TITLE           | nst  | NDEF encoded text                                                                                                                                              |
| NEW_SERVICE_URI             | nsu  | NDEF encoded URI                                                                                                                                               |
| BASKET_PRICE                | bpr  | Nested NDEF message with `mon` and `ccd` records                                                                                                               |
| BASKET_PRICE_AMOUNT         | mon  | Numeric price value                                                                                                                                            |
| BASKET_PRICE_CURRENCY       | ccd  | NDEF encoded text currency code                                                                                                                                |


## Statuses

Status field is returned inside of the session record as the last byte.

| Name                    | Value |
| ----------------------- | ----- |
| UNKNOWN                 | 0x00  |
| OK                      | 0x01  |
| NDEF_FORMAT_INVALID     | 0x02  |
| UNSUPPORTED_VERSION     | 0x03  |
| INVALID_SEQUENCE_NUMBER | 0x04  |
| UNKNOWN_MERCHANT        | 0x05  |
| MERCHANT_INFO_MISSING   | 0x06  |
| SERVICE_DATA_MISSING    | 0x07  |
| RESEND_REQUEST          | 0x08  |
| DATA_NOT_AVAILABLE_YET  | 0x09  |


## Terminal capabilities   

Used in side POS capabilities record. Capabilities are listed like bytes in order from left to right (aka reverse order)

System capabilities:
| Name                   | Value | Effect                                       |
| ---------------------- | ----- | -------------------------------------------- |
| SYSTEM_STANDALONE      | 0x01  |                                              |
| SYSTEM_SEMI_INTEGRATED | 0x02  |                                              |
| SYSTEM_UNATTENDED      | 0x04  |                                              |
| SYSTEM_ONLINE          | 0x08  |                                              |
| SYSTEM_OFFLINE         | 0x10  |                                              |
| SYSTEM_MMP             | 0x20  |                                              |
| SYSTEM_ZLIB_SUPPORTED  | 0x40  | Pass data will be compressed. Adviced to use |

User interface capabilities. No known effects:   

| Name                | Value | Effect |
| ------------------- | ----- | ------ |
| UI_PRINTER          | 0x01  |        |
| UI_PRINTER_GRAPHICS | 0x02  |        |
| UI_DISPLAY          | 0x04  |        |
| UI_IMAGES           | 0x08  |        |
| UI_AUDIO            | 0x10  |        |
| UI_ANIMATION        | 0x20  |        |
| UI_VIDEO            | 0x40  |        |

Checkout capabilities.  No known effects:   

| Name                              | Value | Effect |
| --------------------------------- | ----- | ------ |
| CHECKOUT_SUPPORT_PAYMENT          | 0x01  |        |
| CHECKOUT_SUPPORT_DIGITAL_RECEIPT  | 0x02  |        |
| CHECKOUT_SUPPORT_SERVICE_ISSUANCE | 0x04  |        |
| CHECKOUT_SUPPORT_OTA_POS_DATA     | 0x08  |        |

CVM capabilities.  No known effects:
| Name                      | Value | Effect |
| ------------------------- | ----- | ------ |
| CVM_ONLINE_PIN            | 0x01  |        |
| CVM_CD_PIN                | 0x02  |        |
| CVM_SIGNATURE             | 0x04  |        |
| CVM_NOCVM                 | 0x08  |        |
| CVM_DEVICE_GENERATED_CODE | 0x10  |        |
| CVM_SP_GENERATED_CODE     | 0x20  |        |
| CVM_ID_CAPTURE            | 0x40  |        |
| CVM_BIOMETRIC             | 0x80  |        |

VAS Type capabilities. Set proper type for proper UI interaction:
| Name                  | Value | Effect |
| --------------------- | ----- | ------ |
| TAP_PASS_ONLY         | 0x01  |        |
| TAP_PAYMENT_ONLY      | 0x02  |        |
| TAP_PASS_AND_PAYMENT  | 0x04  |        |
| TAP_PASS_OVER_PAYMENT | 0x08  |        |


## Request object type

Used in service request list:
| Name                      | Value |
| ------------------------- | ----- |
| ALL                       | 0x00  |
| ALL_EXCEPT_PPSE           | 0x01  |
| PPSE                      | 0x02  |
| LOYALTY                   | 0x03  |
| OFFER                     | 0x04  |
| GIFT_CARD                 | 0x05  |
| PRIVATE_LABEL_CARD        | 0x06  |
| EVENT_TICKET              | 0x07  |
| FLIGHT                    | 0x08  |
| TRANSIT                   | 0x09  |
| CLOUD_BASED_WALLET        | 0x10  |
| MOBILE_MARKETING_PLATFORM | 0x11  |
| GENERIC                   | 0x12  |
| WALLET_CUSTOMER           | 0x40  |


## Push Data related options

PUSH DATA seems to be cut or limited in newer versions of SmartTap. This info is provided solely for reference, it has no use IRL.

### Push service type

| Name        | Value |
| ----------- | ----- |
| UNSPECIFIED | 0x00  |
| VALUABLE    | 0x01  |
| RECEIPT     | 0x02  |
| SURVEY      | 0x03  |
| GOODS       | 0x04  |
| SIGNUP      | 0x05  |

### Service status usage

| Name           | Value |
| -------------- | ----- |
| UNDEFINED      | 0x00  |
| SUCCESS        | 0x01  |
| INVALID_FORMAT | 0x02  |
| INVALID_VALUE  | 0x03  |
| UNKNOWN        | 0xff  |

### Service update operation

| Name             | Value |
| ---------------- | ----- |
| NO_OP            | 0x00  |
| REMOVE           | 0x01  |
| SET_BALANCE      | 0x02  |
| ADD_BALANCE      | 0x03  |
| SUBTRACT_BALANCE | 0x04  |
| FREE             | 0x05  |
| UNKNOWN          | 0xFF  |


# Commands

**TODO**

# Notes

- If you find any mistakes/typos or have extra information to add, feel free to raise an issue or create a pull request.
- Reverse-engineering efforts were started way before Google published an open-source implementation example at the end of 2022, which made this project somewhat obsolete.  
  There are some aspects still left uncovered (such as data format, parameters, extra commands), goal of this repo would be to describe blind spots in more detail in the near future ©, plus to provide examples, such as communication logs.


# References


- Google resources:
  - [Google Smart Tap](https://developers.google.com/wallet/smart-tap);
  - [Smart Tap overview](https://developers.google.com/wallet/smart-tap/introduction/overview);
  - [Smart Tap communication flow](https://developers.google.com/wallet/smart-tap/introduction/communication-flow);
  - [Smart Tap example project](https://github.com/google-pay/smart-tap-sample-app). Note that it does not implement the full protocol, for instance "Get more data" and "Push data" commands are missing;
- Analysed applications:
  - [Google Play services](https://play.google.com/store/apps/details?id=com.google.android.gms&hl=en_US);
  - [Google Wallet](https://wallet.google).
- Software analysis tools:
  - [Jadx](https://github.com/skylot/jadx).
