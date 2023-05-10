# Google Smart Tap

<img src="./assets/HEADER.VASONLY.webp" alt="![Smart Tap VAS only]" width=250px>
<img src="./assets/HEADER.VASANDPAY.webp" alt="![Smart Tap VAS and payment]" width=250px>

Google Smart tap is a proprietary NFC protocol that can be used for sending data from a mobile device to an NFC terminal.

Data is conveyed from the device to the terminal in encrypted form, using keys derived during the channel negotiation.

At the moment of negotiation, the reader sends a device its collector id, key version, and a signature of device data signed using the collector key, thus proving to the device that the reader is allowed to get the information.

Only one pass (object) could be conveyed during a single tap. Reading multiple objects related to separate collector ids and keys is not supported, instead one has to add a collector id to needed pass type to enable them being read by their partner.

Version 2.1 was current at the time of writing.


# Application identifiers


Smart Tap can be activated using multiple application ids (AID):

1. Universal VAS AID (hex of encoded 'OSE.VAS.01'), also used by Apple VAS.  
    ```
    4f53452e5641532e3031
    ```
2. Smart Tap 1 (Deprecated)
    ```
    a000000476d0000101
    ```
3. Smart Tap 2
    ```
    a000000476d0000111
    ```


# Commands


As of version 2.1 following commands are available:

1. Negotiate secure channel
2. Get data
3. Get additional data
4. Push data

