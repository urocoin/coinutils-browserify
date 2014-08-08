coinutils-browserify
====================

A fully self-contained client side JavaScript library bundle for creating and using with URO/LTC/BTC brain wallets to sign and receive block chain currency transactions.

Originally designed to be used with BlockCypher's Transactions and Blockchain APIs, but can work with any cryptocurrency workload based on hex string and Buffer encodings.

All the cryptography is done using CryptoCoinJS, see [http://cryptocoinjs.com/]

# Available Functions

```javascript
/**
   * Takes in a hex string message, decodes it into binary, signs it with ECDSA, 
   * and returns the result as a hex string
   * @param {hex string} shaMsg the message to sign
   * @param {Buffer} privateKey the private key to sign with, normally you would 
   * get this from CoinKey.privateKey (from CryptocoinJS)
   */
  this.signMsg =  function (shaMsg, privateKey) {
    var Buffer = require('buffer').Buffer
    var sigRs = ecdsa.sign(new Buffer(shaMsg, 'hex'), privateKey);
    var byteArr = ecdsa.serializeSig(sigRs);
    return new Buffer(byteArr).toString('hex');
  };
    
  /**
   * Takes in a hex string message and verifies it against the 
   * @param {hex string} shaMsg the message to test the signature against
   * @param {hex string} signature the signature 
   * @param {hex string} 
   */
  this.verifyMsg = function (shaMsg, signature, publicKey) {
    var Buffer = require('buffer').Buffer
    var shaMsgBuf = new Buffer(shaMsg, 'hex');
    var sigBuf = new Buffer(signature, 'hex');
    var pubKeyBuf = new Buffer(publicKey, 'hex');
    return ecdsa.verify(shaMsgBuf, sigBuf, pubKeyBuf);
  };
    
  /**
   * Returns a privateKey as a Buffer which is derived deterministically the 
   * username and password arguments
   */
  this.createBrainPrivateKey = function (username, password) {
    return crypto.createHash('sha256').update(username + password).digest();
  };
    
  /**
   * Converts a Buffer to a hex string for sending across the network
   */    
  this.bufferToHex = function (buf) {
    return buf.toString('hex');
  };
    
  /**
   * Creates a CryptocoinJS CoinKey for the URO network corresponding 
   * to the supplied private key 
   * @param {Buffer} privateKey
   */
  this.createUroCoinKey = function (privateKey) {
    return new CoinKey(privateKey, CoinInfo('URO').versions);
  };
    
  /**
   * Creates a CryptocoinJS CoinKey for the LTC network corresponding 
   * to the supplied private key 
   * @param {Buffer} privateKey
   */
  this.createLtcCoinKey = function (privateKey) {
    return new CoinKey(privateKey, CoinInfo('LTC').versions);
  };
    
  /**
   * Creates a CryptocoinJS CoinKey for the BTC network corresponding 
   * to the supplied private key 
   * @param {Buffer} privateKey
   */
  this.createBtcCoinKey = function (privateKey) {
    return new CoinKey(privateKey, CoinInfo('BTC').versions);
  };
```
