# Google Smart Tap

<img src="./assets/HEADER.VASONLY.webp" alt="![Smart Tap VAS only]" width=250px>
<img src="./assets/HEADER.VASANDPAY.webp" alt="![Smart Tap VAS and payment]" width=250px>

Google Smart tap is a proprietary NFC protocol that can be used for sending data from a mobile device to an NFC terminal.

Smart Tap has gone through multiple versions throughout its life. As of writing date the version of this protocol is 2.1. Data can only be read after seucre channel negotiation and only in encrypted form.


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

