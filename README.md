coinutils-browserify
====================

A fully self-contained client side JavaScript library bundle for creating and using URO/LTC/BTC brain wallets to sign and verify blockchain currency transactions.

Originally designed to be used with the [BlockCypher](http://blockcypher.com/) Transactions and Blockchain APIs, but can work with any cryptocurrency transactions workload based on hex string and Buffer formats.

All the cryptography is done using [CryptoCoinJS](http://cryptocoinjs.com/).

#### Available Functions

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

#### Sample code demonstrating use with BlockCypher V1 API

```javascript
function checkUserPass(){
    var username = $('#uro-username-input').val();
    if (username.length < 8) {
        alert('Username needs to be 8 characters or more');
        return false;
    }
    var password = $('#uro-password-input').val();
    if (username.length < 8) {
        alert('Password needs to be 10 characters or more');
        return false;
    }    
    Nuro.username = username;
    Nuro.password = password;
    var coinutils = require('coinutils.js');
    Nuro.privateKey = coinutils.createBrainPrivateKey(username, password);
    Nuro.uroCoinKey = coinutils.createUroCoinKey(Nuro.privateKey);
    Nuro.ltcCoinKey = coinutils.createLtcCoinKey(Nuro.privateKey);
    Nuro.btcCoinKey = coinutils.createBtcCoinKey(Nuro.privateKey);

    return true;
}

function switchSubPage(pageId) {
    if (checkUserPass()) {
        activate_subpage('#' + pageId);
    } else {
        activate_subpage('#change-account');
    }
}

function updateReceivePageContent() {
    $("#uro-receive-address-lbl").text(Nuro.uroCoinKey.publicAddress);
    Nuro.qrcode.clear();
    Nuro.qrcode.makeCode(Nuro.uroCoinKey.publicAddress);
}

/*
Reset the send page to clean state
*/
function resetSendPageStatus() {
    $('#recipient-address-input').val('');
    $('#send-amount-input').val('');
    $('#send-status-label').html('Please scan/paste in valid receive address and enter the amount to send');
}

function checkError(msg) {
    if (msg.errors && msg.errors.length) {
        log("Errors occured!!/n" + msg.errors.join("/n"));
        return true;
    }
}

// 1. Post our simple transaction information to get back the fully built transaction,
//    includes fees when required.
// amount should be in number of coins, e.g. 1.2, or 0.1
function createTx(srcAddr, destAddr, amount) {
    var rootUrl = "https://api.blockcypher.com/v1/uro/main";
    var newTx = {
        "inputs": [{"addresses": [srcAddr]}],
        "outputs": [{"addresses": [destAddr], "value": amount * Nuro.satoshiToCoinMultiplier}]
    };
    $.post(rootUrl+"/txs/new", JSON.stringify(newTx), function(newTx){
        if (!checkError(newTx)) {
            $('#send-status-label').html('Sending funds. Please wait...');
        } else {
            $('#send-status-label').html('Error occured during processing transaction');
        }
        signAndSendTx(newTx);
    });
}

// 2. Sign the hexadecimal strings returned with the fully built transaction and include
//    the source public address.
function signAndSendTx(newTx) {
    var rootUrl = "https://api.blockcypher.com/v1/uro/main";
    if (checkError(newTx)) return;
    var coinutils = require('coinutils.js');
    var publicKey = Nuro.uroCoinKey.publicKey;
    var privateKey = Nuro.uroCoinKey.privateKey;
    newTx.pubkeys = [];

    newTx.signatures = newTx.tosign.map(function(tosign) {
        newTx.pubkeys.push(coinutils.bufferToHex(publicKey));
        return coinutils.signMsg(tosign, privateKey);
    });
    console.log(JSON.stringify(newTx));
    $.post(rootUrl+"/txs/send", JSON.stringify(newTx), function(newTx){
        if (!checkError(newTx)) {
            $('#send-status-label').html('Funds sent sucessfully.');
            alert('Funds sent. The recipent will be able to confirm receipt of the funds after a few minutes');
            resetSendPageStatus();
        } else {
            $('#send-status-label').html('Error occured during processing transaction');
        }
    });
}

/*
Sends a transaction from data entered on the send page
*/
function sendTxFromPageData() {
    var destAddr = $('#recipient-address-input').val();
    var amt = $('#send-amount-input').val();
    createTx(Nuro.uroCoinKey.publicAddress, destAddr, amt);
}

function satoshiToCoinStr(sats) {
    var coins = "";
    if (sats == 0) {
        coins = 0;
    } else {
        coins = sats / Nuro.satoshiToCoinMultiplier;
    }
    return coins + " " + Nuro.currentCoinTicker;
}

```
